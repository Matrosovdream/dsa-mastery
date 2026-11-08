# Step 02 — Environment & Toolkit · 🟡 Medium

Examples **7–11**: the empty control, the size sweep, `benchstat`, `-race`, and fuzzing.

These are the four tools that find what a table-driven test cannot: a benchmark that measured nothing,
a complexity you assumed, a race that hides behind a passing test, and an input you never imagined.

> ⚠️ Examples 7–9 print timings — **sample output**, produced on an Apple M4 with Go 1.26.3. Example
> 10's race report contains addresses and goroutine IDs that change every run. Example 11's fuzzing
> is nondeterministic: you'll get a different counterexample and a different corpus filename.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`). Unlike the easy tier,
all five twins here are **plain programs, not test files**: they report measurements (a control, a
sweep, a statistical comparison, a race, a shrinking trace), and libtest swallows the output of passing
tests. Build them with `-O`:

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go package and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. `benchstat`, `-race` and `go test -fuzz` have no std counterpart,
so each twin says what it does instead and names the real Rust tool.

> ← Back to the [index](README.md) · Previous tier: [🟢 easy](1-easy.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. The empty control

`🟡 medium` · *Benchmark hygiene*

Lesson 01 proved that `b.Loop` has a per-iteration cost of its own. This is that lesson turned into a
habit: **ship an empty benchmark in every package**. It measures nothing on purpose, and it is the
floor below which no other benchmark in that package can be believed.

Watch what happens: `BenchmarkSquare` reports **1.607 ns/op** and the empty control reports
**1.654 ns/op**. The "measurement" of `Square` is *below* the cost of the loop that measures it —
which means it is noise, not a result.

**Steps:**

1. Write `BenchmarkEmptyControl` with an empty `b.Loop` body.
2. Write the benchmark you actually care about.
3. Compare. If they're within noise, batch the work and divide.

**`square.go`**

```go
package algo

// Square is one multiply -- far cheaper than the benchmark harness that measures it.
func Square(x int) int { return x * x }
```

**`square_test.go`**

```go
package algo

import "testing"

// Ship this benchmark in every package. It measures nothing, on purpose: it is
// the floor below which no other benchmark in this file can be believed.
func BenchmarkEmptyControl(b *testing.B) {
	for b.Loop() {
	}
}

// Looks like it measures Square. Compare it to the control before believing it.
func BenchmarkSquare(b *testing.B) {
	i := 0
	for b.Loop() {
		Square(i)
		i++
	}
}

// The fix: make one "op" big enough that the harness cost is noise. Divide the
// reported ns/op by 1000 yourself to get the per-call number.
func BenchmarkSquareBatch1000(b *testing.B) {
	xs := make([]int, 1000)
	for i := range xs {
		xs[i] = i
	}
	s := 0
	for b.Loop() {
		for _, v := range xs {
			s += Square(v)
		}
	}
	// Keep the accumulated value observable so the loop cannot be deleted.
	if s == 0 {
		b.Fatal("unreachable, but the compiler does not know that")
	}
}
```

**Run it:**

```bash
go test -bench=. -run='^$' ./ex07
```

**Sample output:**

```
goos: darwin
goarch: arm64
pkg: l02/ex07
cpu: Apple M4
BenchmarkEmptyControl-10       	635776324	         1.654 ns/op
BenchmarkSquare-10             	747192772	         1.607 ns/op
BenchmarkSquareBatch1000-10    	 1548980	       774.7 ns/op
PASS
ok  	l02/ex07	3.973s
```

`774.7 / 1000 = 0.77 ns` per call — the only real number in the table, and less than half of what the
naive benchmark "measured".

**Rust:**

```rust
// Stable Rust has no built-in benchmark runner (`#[bench]` is nightly-only), so
// this is the same std-only harness as example 6: a calibrating loop that plays
// the role of `b.Loop`. The control matters even more here, because in Rust the
// optimizer can delete an empty loop outright (see `EmptyBody` below).
use std::hint::black_box;
use std::time::{Duration, Instant};

/// Square is one multiply -- far cheaper than the benchmark harness that measures it.
/// (Go's `x * x` wraps silently; `wrapping_mul` says so out loud.)
fn square(x: i64) -> i64 {
    x.wrapping_mul(x)
}

/// Calibrates like b.Loop: grows the iteration count until a run takes
/// ~200 ms. Returns (iterations of the final run, ns per iteration).
fn ns_per_op(mut f: impl FnMut()) -> (u64, f64) {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return (n, el.as_nanos() as f64 / n as f64);
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

/// One benchmark row: name, iterations, ns/op. Returns ns/op for the caller.
fn bench(name: &str, f: impl FnMut()) -> f64 {
    let (iters, ns) = ns_per_op(f);
    println!("{name:<22}{iters:>16}{ns:>10.3} ns/op");
    ns
}

fn main() {
    println!("os: {}", std::env::consts::OS);
    println!("arch: {}", std::env::consts::ARCH);

    // Ship this one in every suite. It measures nothing, on purpose: it is the
    // floor below which no other row can be believed. `black_box(())` is an
    // optimization barrier that emits no instructions, so the loop survives
    // but does nothing else.
    let control = bench("EmptyControl", || {
        black_box(());
    });

    // The Rust-only trap: a truly empty body. LLVM sees a loop that does
    // nothing and removes it, so the "measurement" is exactly zero and the
    // iteration count runs off to the calibration cap. This is the opposite
    // failure to Go's (a floor that is too high), and the same cure: look at
    // the control before you believe the other rows.
    bench("EmptyBody", || {});

    // Looks like it measures `square`. Compare it to the control before believing it.
    let mut i = 0i64;
    let naive = bench("Square", || {
        black_box(square(black_box(i)));
        i += 1;
    });

    // The fix: make one "op" big enough that the harness cost is noise. Divide the
    // reported ns/op by 1000 yourself to get the per-call number.
    let xs: Vec<i64> = (0..1000).collect();
    let batch = bench("SquareBatch1000", || {
        let mut s = 0i64;
        for &v in black_box(&xs) {
            s += square(v);
        }
        // Keep the accumulated value observable so the loop cannot be deleted.
        black_box(s);
    });

    println!();
    println!(
        "Square vs the control:  {:+.1}%",
        (naive / control - 1.0) * 100.0
    );
    println!("SquareBatch1000 / 1000 = {:.3} ns per call", batch / 1000.0);
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O square.rs -o square && ./square
```

**Rust sample output:**

```
os: macos
arch: aarch64
EmptyControl                 911895990     0.268 ns/op
EmptyBody              100000000000000     0.000 ns/op
Square                       864031969     0.304 ns/op
SquareBatch1000                1275516   194.168 ns/op

Square vs the control:  +13.5%
SquareBatch1000 / 1000 = 0.194 ns per call
```

> **Go vs Rust:** The control did its job, but the floor is far lower than Go's: 0.25 to 0.35 ns per iteration in every run here (ten runs, on a machine other jobs were using) against Go's 1.654 ns `b.Loop`, and `Square` came out only 0.01 to 0.1 ns above it, where Go's came out 0.05 ns below it, so either way the row reports the harness, not the multiply. A literally empty body (`EmptyBody`) is a Rust-only trap: LLVM deletes the loop, which gives 0.000 ns/op and an iteration count that ran to the calibration cap, so the control has to be pinned with `black_box(())`. Batching 1000 calls gave 0.17 to 0.26 ns per call in those runs against Go's 0.77 ns; in a standalone copy of the loop the compiler unrolled it 4x with four independent multiply-add accumulators (not SIMD), a plausible reason, but I did not disassemble the benchmark binary itself.

**Takeaway:** any benchmark within noise of the empty control is a benchmark of the empty control. Batch until the work dominates.

---

## 8. The size sweep

`🟡 medium` · *Sub-benchmarks / reading complexity off a benchmark*

A single ns/op number tells you nothing about an algorithm — complexity is a statement about *growth*,
so you have to vary n to see it. `b.Run` makes sub-benchmarks; the naming convention
`name/n=1000` keeps the output readable and greppable.

Read the two columns below and you can classify both algorithms without looking at their source.

**Steps:**

1. Loop over sizes on the outside, `b.Run` on the inside.
2. Search for the **last** element, so the linear scan is measured at its worst case.
3. Compare each row to the one above: what does 8× the input cost?

**`search.go`**

```go
package algo

// LinearSearch scans every element: O(n).
func LinearSearch(xs []int, target int) int {
	for i, v := range xs {
		if v == target {
			return i
		}
	}
	return -1
}

// BinarySearch halves the range each step: O(log n). xs must be sorted.
func BinarySearch(xs []int, target int) int {
	lo, hi := 0, len(xs)-1
	for lo <= hi {
		mid := lo + (hi-lo)/2 // not (lo+hi)/2 -- that can overflow
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
```

**`search_test.go`**

```go
package algo

import (
	"fmt"
	"testing"
)

// One benchmark function, one sub-benchmark per (algorithm, size) pair. The
// sweep is the point: a single ns/op number tells you nothing about growth.
func BenchmarkSearch(b *testing.B) {
	for _, n := range []int{64, 512, 4096, 32768, 262144} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i * 2
		}
		target := (n - 1) * 2 // the last element: worst case for the linear scan

		b.Run(fmt.Sprintf("linear/n=%d", n), func(b *testing.B) {
			for b.Loop() {
				LinearSearch(xs, target)
			}
		})
		b.Run(fmt.Sprintf("binary/n=%d", n), func(b *testing.B) {
			for b.Loop() {
				BinarySearch(xs, target)
			}
		})
	}
}
```

**Run it:**

```bash
go test -bench=. -run='^$' ./ex08
```

**Sample output:**

```
goos: darwin
goarch: arm64
pkg: l02/ex08
cpu: Apple M4
BenchmarkSearch/linear/n=64-10         	57249464	        20.73 ns/op
BenchmarkSearch/binary/n=64-10         	320650563	         3.750 ns/op
BenchmarkSearch/linear/n=512-10        	 9574267	       125.6 ns/op
BenchmarkSearch/binary/n=512-10        	192973693	         6.236 ns/op
BenchmarkSearch/linear/n=4096-10       	 1266687	       946.3 ns/op
BenchmarkSearch/binary/n=4096-10       	133159178	         9.061 ns/op
BenchmarkSearch/linear/n=32768-10      	  162292	      7581 ns/op
BenchmarkSearch/binary/n=32768-10      	100000000	        11.36 ns/op
BenchmarkSearch/linear/n=262144-10     	   19742	     60921 ns/op
BenchmarkSearch/binary/n=262144-10     	80904103	        14.66 ns/op
```

Every row is 8× the n of the row two above it:

- **linear:** 20.73 → 125.6 → 946.3 → 7581 → 60921. Each step is ~8× the time. That is O(n).
- **binary:** 3.75 → 6.24 → 9.06 → 11.36 → 14.66. Each step adds a flat **~3 ns**, because 8× the input is 3 more halvings. That is O(log n).

At n=262144 the gap is **4155×**, and it keeps widening forever.

**Rust:**

```rust
// Same std-only harness as examples 6 and 7 (stable Rust has no `testing.B`;
// `cargo bench` with criterion or divan is the real tool and is not run here).
// Go's b.Run("linear/n=64", ...) becomes one `bench` call per (algorithm, size)
// pair, named the same way so the output stays readable and greppable.
use std::cmp::Ordering;
use std::hint::black_box;
use std::time::{Duration, Instant};

/// Scans every element: O(n).
fn linear_search(xs: &[i64], target: i64) -> Option<usize> {
    xs.iter().position(|&v| v == target)
}

/// Halves the range each step: O(log n). `xs` must be sorted.
fn binary_search(xs: &[i64], target: i64) -> Option<usize> {
    // Half-open range [lo, hi). Go's inclusive `hi = len-1` form would need
    // `hi = mid - 1`, which underflows a usize when mid is 0.
    let (mut lo, mut hi) = (0, xs.len());
    while lo < hi {
        let mid = lo + (hi - lo) / 2; // not (lo+hi)/2 -- that can overflow
        match xs[mid].cmp(&target) {
            Ordering::Equal => return Some(mid),
            Ordering::Less => lo = mid + 1,
            Ordering::Greater => hi = mid,
        }
    }
    None
}

/// Calibrates like b.Loop: grows the iteration count until a run takes
/// ~200 ms. Returns (iterations of the final run, ns per iteration).
fn ns_per_op(mut f: impl FnMut()) -> (u64, f64) {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return (n, el.as_nanos() as f64 / n as f64);
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

/// One benchmark row: name, iterations, ns/op. Returns ns/op for the caller.
fn bench(name: &str, f: impl FnMut()) -> f64 {
    let (iters, ns) = ns_per_op(f);
    println!("{name:<24}{iters:>12}{ns:>12.3} ns/op");
    ns
}

// One loop over sizes, one row per (algorithm, size) pair. The sweep is the
// point: a single ns/op number tells you nothing about growth.
fn main() {
    println!("os: {}", std::env::consts::OS);
    println!("arch: {}", std::env::consts::ARCH);

    let sizes = [64usize, 512, 4096, 32768, 262144];
    let (mut linear, mut binary) = (Vec::new(), Vec::new());
    for n in sizes {
        let xs: Vec<i64> = (0..n as i64).map(|i| i * 2).collect();
        let target = (n as i64 - 1) * 2; // the last element: worst case for the linear scan

        linear.push(bench(&format!("search/linear/n={n}"), || {
            black_box(linear_search(black_box(&xs), black_box(target)));
        }));
        binary.push(bench(&format!("search/binary/n={n}"), || {
            black_box(binary_search(black_box(&xs), black_box(target)));
        }));
    }

    // Read growth off the rows: each step is 8x the input. Multiplicative
    // growth is O(n); a flat additive step is O(log n).
    println!();
    println!("{:<16}{:>12}{:>16}", "8x the input", "linear", "binary");
    for k in 1..sizes.len() {
        println!(
            "{:<16}{:>11.1}x{:>13.2} ns",
            format!("{} -> {}", sizes[k - 1], sizes[k]),
            linear[k] / linear[k - 1],
            binary[k] - binary[k - 1]
        );
    }
    let last = sizes.len() - 1;
    println!(
        "\nat n={}: linear / binary = {:.0}x",
        sizes[last],
        linear[last] / binary[last]
    );
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O search.rs -o search && ./search
```

**Rust sample output:**

```
os: macos
arch: aarch64
search/linear/n=64          13634233      17.424 ns/op
search/binary/n=64          67011796       3.656 ns/op
search/linear/n=512          1643065     147.444 ns/op
search/binary/n=512         42760740       6.562 ns/op
search/linear/n=4096          223245    1195.559 ns/op
search/binary/n=4096        24733980       8.852 ns/op
search/linear/n=32768          27127    8904.856 ns/op
search/binary/n=32768       22311486      11.010 ns/op
search/linear/n=262144          3476   69510.824 ns/op
search/binary/n=262144      18663850      13.190 ns/op

8x the input          linear          binary
64 -> 512               8.5x         2.91 ns
512 -> 4096             8.1x         2.29 ns
4096 -> 32768           7.4x         2.16 ns
32768 -> 262144         7.8x         2.18 ns

at n=262144: linear / binary = 5270x
```

> **Go vs Rust:** Same shape as Go, with `Option<usize>` instead of `-1` and a half-open `[lo, hi)` range, because the inclusive `hi = mid - 1` would underflow a `usize` at 0. Linear grew roughly 8x per 8x of n (steps of 5.7x to 10.1x over six runs) while binary went from 3.6-4.9 ns at n=64 to 12.8-17 ns at n=262144, so 4096x more input cost only about 3 to 4x the time; the final linear/binary gap was 4800x to 5600x against Go's 4155x. Binary's individual 8x steps were noisy on this shared machine (0.15 to 5.4 ns), so only the overall flatness is a robust claim, and I do not compare absolute ns between the two languages.

**Takeaway:** you can measure a complexity class. Sweep n by a constant factor and look at what each step costs — multiplicative for O(n), additive for O(log n).

---

## 9. Before and after, with statistics

`🟡 medium` · *benchstat + build tags*

Two ns/op numbers are not a comparison — benchmarks are noisy, and "it looks faster" is how people
ship regressions. `benchstat` runs the statistics: median, variation, percentage delta, and a p-value
saying whether the difference is real.

The mechanism worth stealing here is the **build tag**. Both versions of `Grow` have the same name in
the same package, selected at build time, so there is one benchmark name for benchstat to match on —
and the test suite proves both versions are still correct.

**Install it once:**

```bash
go install golang.org/x/perf/cmd/benchstat@latest
```

**Steps:**

1. Put the two implementations in tag-guarded files (`!prealloc` and `prealloc`).
2. Run each `-count=10` into its own file.
3. Compare with `benchstat old.txt new.txt`.

**`grow_nil.go`**

```go
//go:build !prealloc

package algo

// Grow builds the first n integers without telling append how many are coming.
// This is the "before" version -- build it with no tags.
func Grow(n int) []int {
	var out []int
	for i := 0; i < n; i++ {
		out = append(out, i)
	}
	return out
}
```

**`grow_prealloc.go`**

```go
//go:build prealloc

package algo

// Grow builds the first n integers, reserving the capacity up front.
// This is the "after" version -- build it with -tags=prealloc.
func Grow(n int) []int {
	out := make([]int, 0, n)
	for i := 0; i < n; i++ {
		out = append(out, i)
	}
	return out
}
```

**`grow_test.go`**

```go
package algo

import "testing"

// Both builds must pass this: the optimization is only allowed to change speed.
func TestGrow(t *testing.T) {
	got := Grow(5)
	if len(got) != 5 {
		t.Fatalf("len = %d, want 5", len(got))
	}
	for i, v := range got {
		if v != i {
			t.Errorf("got[%d] = %d, want %d", i, v, i)
		}
	}
}

// One benchmark, one name -- benchstat matches the two runs by this name.
func BenchmarkGrow(b *testing.B) {
	b.ReportAllocs()
	for b.Loop() {
		Grow(10000)
	}
}
```

**Run it:**

```bash
go test -bench=Grow -benchmem -count=10 -run='^$' ./ex09 > old.txt
go test -tags=prealloc -bench=Grow -benchmem -count=10 -run='^$' ./ex09 > new.txt
benchstat old.txt new.txt
```

**Sample output:**

```
goos: darwin
goarch: arm64
pkg: l02/ex09
cpu: Apple M4
        │   old.txt    │               new.txt               │
        │    sec/op    │   sec/op     vs base                │
Grow-10   25.471µ ± 2%   5.647µ ± 1%  -77.83% (p=0.000 n=10)

        │    old.txt    │               new.txt                │
        │     B/op      │     B/op      vs base                │
Grow-10   349.24Ki ± 0%   80.00Ki ± 0%  -77.09% (p=0.000 n=10)

        │   old.txt   │              new.txt               │
        │  allocs/op  │ allocs/op   vs base                │
Grow-10   19.000 ± 0%   1.000 ± 0%  -94.74% (p=0.000 n=10)
```

Reading it: `± 2%` is the variation across the 10 runs, `-77.83%` is the change, and `p=0.000` says
the difference is statistically significant. If benchstat prints `~` instead of a percentage, the two
sides are indistinguishable and your "optimization" did nothing.

**Rust:**

```rust
// `benchstat` is a Go tool and is not installed here, so nothing below is its output.
// What a std-only port CAN do is the same procedure: take 10 samples of the "before"
// and 10 of the "after" (Go's -count=10), then report median, spread, % change and
// a p-value (benchstat uses the Mann-Whitney U test; so does `mann_whitney_p`).
//
// Go selects ONE `Grow` per build with `//go:build prealloc`. Rust's counterpart is
// `#[cfg(prealloc)]` with `rustc --cfg prealloc` (or a cargo feature). A build-time switch needs
// two builds, and this run must show both sides, so both versions are compiled in
// side by side as modules and each is measured under the same label, `Grow`.
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

/// The "before": builds the first n integers without telling the Vec how many are coming.
mod grow_nil {
    pub fn grow(n: usize) -> Vec<i64> {
        let mut out = Vec::new();
        for i in 0..n as i64 {
            out.push(i);
        }
        out
    }
}

/// The "after": reserves the capacity up front.
mod grow_prealloc {
    pub fn grow(n: usize) -> Vec<i64> {
        let mut out = Vec::with_capacity(n);
        for i in 0..n as i64 {
            out.push(i);
        }
        out
    }
}

/// Both versions must pass this: the optimization is only allowed to change speed.
fn check_grow(grow: fn(usize) -> Vec<i64>) {
    let got = grow(5);
    assert_eq!(got, [0, 1, 2, 3, 4]);
}

// ---- allocation counter: stand-in for -benchmem / b.ReportAllocs() ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)
static BYTES: AtomicUsize = AtomicUsize::new(0); // bytes requested, summed over all events

unsafe impl GlobalAlloc for Counting {
    // `unsafe` is unavoidable: GlobalAlloc is an unsafe trait. Every method only
    // counts and then forwards to the system allocator unchanged.
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

/// (allocation events, bytes requested) for ONE call of `f`.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, usize) {
    let (a0, b0) = (ALLOCS.load(Relaxed), BYTES.load(Relaxed));
    drop(black_box(f()));
    (ALLOCS.load(Relaxed) - a0, BYTES.load(Relaxed) - b0)
}

/// Calibrates like b.Loop: grows the iteration count until a run takes
/// ~200 ms. Returns (iterations of the final run, ns per iteration).
fn ns_per_op(mut f: impl FnMut()) -> (u64, f64) {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return (n, el.as_nanos() as f64 / n as f64);
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

/// The three columns of one set of samples, one entry per run.
#[derive(Default)]
struct Samples {
    ns: Vec<f64>,
    bytes: Vec<f64>,
    allocs: Vec<f64>,
}

/// `go test -bench=Grow -benchmem -count=10`: ten runs, printed the way `go test` prints them.
fn run_count(label: &str, grow: fn(usize) -> Vec<i64>, count: usize) -> Samples {
    println!("{label}");
    let mut s = Samples::default();
    for _ in 0..count {
        let (allocs, bytes) = allocs_of(|| grow(black_box(10_000)));
        let (iters, ns) = ns_per_op(|| {
            black_box(grow(black_box(10_000)));
        });
        println!("Grow {iters:>12}{ns:>12.1} ns/op{bytes:>9} B/op{allocs:>4} allocs/op");
        s.ns.push(ns);
        s.bytes.push(bytes as f64);
        s.allocs.push(allocs as f64);
    }
    println!();
    s
}

fn median(xs: &[f64]) -> f64 {
    let mut v = xs.to_vec();
    v.sort_by(f64::total_cmp);
    let mid = v.len() / 2;
    if v.len() % 2 == 1 {
        v[mid]
    } else {
        (v[mid - 1] + v[mid]) / 2.0
    }
}

/// Half the range across the runs, as a percentage of the median. This is NOT
/// benchstat's `±` (a confidence interval); it is the simplest honest spread.
fn spread_pct(xs: &[f64]) -> f64 {
    let (lo, hi) = xs
        .iter()
        .fold((f64::MAX, f64::MIN), |(l, h), &x| (l.min(x), h.max(x)));
    (hi - lo) / 2.0 / median(xs) * 100.0
}

/// Exact two-sided Mann-Whitney U test: the chance that two samples drawn from
/// one distribution would be at least this far apart. Counts, for every
/// arrangement of the pooled values, how many (a, b) pairs have a > b, using
///   count(m, n, u) = count(m-1, n, u-n) + count(m, n-1, u)
/// (the largest value is either in `a`, beating all n values of `b`, or in `b`).
fn mann_whitney_p(a: &[f64], b: &[f64]) -> f64 {
    assert!(
        a.iter().all(|x| b.iter().all(|y| x != y)),
        "ties between the samples are not handled"
    );
    let (m, n) = (a.len(), b.len());
    let u = a
        .iter()
        .map(|x| b.iter().filter(|y| x > y).count())
        .sum::<usize>();

    let mut counts = vec![vec![vec![1u64]; n + 1]; m + 1];
    for i in 1..=m {
        for j in 1..=n {
            let mut c = vec![0u64; i * j + 1];
            for (k, slot) in c.iter_mut().enumerate() {
                let beats_all = if k >= j {
                    counts[i - 1][j].get(k - j)
                } else {
                    None
                };
                *slot =
                    beats_all.copied().unwrap_or(0) + counts[i][j - 1].get(k).copied().unwrap_or(0);
            }
            counts[i][j] = c;
        }
    }
    let dist = &counts[m][n];
    let total: u64 = dist.iter().sum();
    let at_most: u64 = dist[..=u].iter().sum();
    let at_least: u64 = dist[u..].iter().sum();
    (2.0 * at_most.min(at_least) as f64 / total as f64).min(1.0)
}

/// One row of the table: median ± spread for each side, then the change.
fn row(name: &str, old: &[f64], new: &[f64], fmt: fn(f64) -> String) {
    let side = |xs: &[f64]| format!("{} ± {:.0}%", fmt(median(xs)), spread_pct(xs));
    let p = mann_whitney_p(old, new);
    let delta = if p > 0.05 {
        "~".to_string() // not distinguishable from noise
    } else {
        format!("{:+.2}%", (median(new) / median(old) - 1.0) * 100.0)
    };
    println!(
        "{name:<11}{:<17}{:<16}{delta} (p={p:.3} n={})",
        side(old),
        side(new),
        old.len()
    );
}

fn main() {
    // The test suite proves both versions are still correct.
    check_grow(grow_nil::grow);
    check_grow(grow_prealloc::grow);
    println!("TestGrow: ok for both versions\n");

    let old = run_count("== old (Go: > old.txt)", grow_nil::grow, 10);
    let new = run_count(
        "== new (Go: -tags=prealloc > new.txt)",
        grow_prealloc::grow,
        10,
    );

    println!("{:<11}{:<17}{:<16}vs base", "Grow", "old", "new");
    row("sec/op", &old.ns, &new.ns, |ns| format!("{:.3}µ", ns / 1e3));
    row("B/op", &old.bytes, &new.bytes, |b| {
        format!("{:.2}Ki", b / 1024.0)
    });
    row("allocs/op", &old.allocs, &new.allocs, |a| format!("{a:.3}"));
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O grow.rs -o grow && ./grow
```

**Rust sample output:**

```
TestGrow: ok for both versions

== old (Go: > old.txt)
Grow        34998      6374.8 ns/op   262112 B/op  13 allocs/op
Grow        36469      6360.9 ns/op   262112 B/op  13 allocs/op
Grow        38236      6293.4 ns/op   262112 B/op  13 allocs/op
Grow        38434      6467.1 ns/op   262112 B/op  13 allocs/op
Grow        32898      6320.6 ns/op   262112 B/op  13 allocs/op
Grow        38273      6168.8 ns/op   262112 B/op  13 allocs/op
Grow        38774      6204.1 ns/op   262112 B/op  13 allocs/op
Grow        39138      6616.6 ns/op   262112 B/op  13 allocs/op
Grow        34676      6311.4 ns/op   262112 B/op  13 allocs/op
Grow        36824      6199.2 ns/op   262112 B/op  13 allocs/op

== new (Go: -tags=prealloc > new.txt)
Grow        87175      2794.5 ns/op    80000 B/op   1 allocs/op
Grow        85634      2785.4 ns/op    80000 B/op   1 allocs/op
Grow        84818      2874.6 ns/op    80000 B/op   1 allocs/op
Grow        71725      3275.8 ns/op    80000 B/op   1 allocs/op
Grow        85779      2887.0 ns/op    80000 B/op   1 allocs/op
Grow        83393      2853.0 ns/op    80000 B/op   1 allocs/op
Grow        84621      2781.6 ns/op    80000 B/op   1 allocs/op
Grow        88916      3335.7 ns/op    80000 B/op   1 allocs/op
Grow        79987      2782.9 ns/op    80000 B/op   1 allocs/op
Grow        74066      3105.1 ns/op    80000 B/op   1 allocs/op

Grow       old              new             vs base
sec/op     6.316µ ± 4%      2.864µ ± 10%    -54.66% (p=0.000 n=10)
B/op       255.97Ki ± 0%    78.12Ki ± 0%    -69.48% (p=0.000 n=10)
allocs/op  13.000 ± 0%      1.000 ± 0%      -92.31% (p=0.000 n=10)
```

> **Go vs Rust:** `benchstat` is not installed here, so the table is this program's own report, not its output: median, a plain half-range spread (not benchstat's confidence interval) and an exact Mann-Whitney p-value (checked against brute-force enumeration of every pairing on seven synthetic sample pairs); `cargo bench` with criterion is the usual Rust route to the real statistics and was not run. Go's build tag becomes `#[cfg(prealloc)]` with `rustc --cfg prealloc` (or a cargo feature), but a build-time switch shows one side per run, so both versions are compiled in as modules and measured under the same `Grow` label. The direction matches Go but the effect is smaller: preallocating cut time by about half (47% to 58% over six runs; the lowest came from a run with a 38% spread) against Go's -77.83%, and allocation events from 13 to 1 (capacities 4, 8, ..., 16384, 262,112 bytes requested) against Go's 19 to 1, so there is less to save; the 'before' also ran in about 6 µs against Go's 25 µs, though not necessarily on the same load or machine.

**Takeaway:** `-count=10` and benchstat, or it didn't happen. One knowing `make([]T, 0, n)` cut 78% of the time and 95% of the allocations.

---

## 10. The race a passing test cannot see

`🟡 medium` · *go test -race*

The function below reduces into a shared variable from four goroutines. It is unambiguously wrong.
Its test **passed 5 times out of 5** — the race window is small, and the answer usually comes out
right anyway. That is what makes concurrency bugs so expensive: they pass CI for months, then corrupt
data under production load.

`go test -race` finds it on the first run, and names the two lines involved.

**Steps:**

1. Run the test normally, several times. Watch it pass.
2. Run it with `-race`.
3. Read the fix: no mutex — each worker gets a private accumulator, combined once at the end.

**`parallel.go`**

```go
package algo

import "sync"

// MaxParallelRacy splits xs across workers and reduces into a shared variable.
// The reduction is unsynchronized: two goroutines can read `best`, both decide
// they are larger, and one write is lost.
//
// It usually returns the right answer anyway -- which is exactly the problem.
func MaxParallelRacy(xs []int, workers int) int {
	best := xs[0]
	chunk := (len(xs) + workers - 1) / workers

	var wg sync.WaitGroup
	for w := 0; w < workers; w++ {
		lo := w * chunk
		hi := min(lo+chunk, len(xs))
		if lo >= hi {
			continue
		}
		wg.Add(1)
		go func(part []int) {
			defer wg.Done()
			for _, v := range part {
				if v > best { // <-- DATA RACE: read and write of a shared int
					best = v
				}
			}
		}(xs[lo:hi])
	}
	wg.Wait()
	return best
}

// MaxParallel is the fix: each worker reduces its own chunk into a private
// variable, and the results are combined once, in one goroutine. No shared
// mutable state means no lock and no race.
func MaxParallel(xs []int, workers int) int {
	chunk := (len(xs) + workers - 1) / workers
	partials := make([]int, workers)

	var wg sync.WaitGroup
	for w := 0; w < workers; w++ {
		lo := w * chunk
		hi := min(lo+chunk, len(xs))
		if lo >= hi {
			partials[w] = xs[0]
			continue
		}
		wg.Add(1)
		go func(w int, part []int) {
			defer wg.Done()
			local := part[0] // only this goroutine touches partials[w]
			for _, v := range part[1:] {
				if v > local {
					local = v
				}
			}
			partials[w] = local
		}(w, xs[lo:hi])
	}
	wg.Wait()

	best := partials[0]
	for _, v := range partials[1:] {
		if v > best {
			best = v
		}
	}
	return best
}
```

**`parallel_test.go`**

```go
package algo

import "testing"

func input() []int {
	xs := make([]int, 10000)
	for i := range xs {
		xs[i] = i
	}
	return xs
}

// This test passes. Run it with -race and it fails anyway.
func TestMaxParallelRacy(t *testing.T) {
	if got := MaxParallelRacy(input(), 4); got != 9999 {
		t.Errorf("MaxParallelRacy = %d, want 9999", got)
	}
}

func TestMaxParallel(t *testing.T) {
	if got := MaxParallel(input(), 4); got != 9999 {
		t.Errorf("MaxParallel = %d, want 9999", got)
	}
}
```

**Run it normally, five times:**

```bash
for i in 1 2 3 4 5; do go test -count=1 ./ex10 | tail -1; done
```

```
ok  	l02/ex10	0.341s
ok  	l02/ex10	0.136s
ok  	l02/ex10	0.108s
ok  	l02/ex10	0.106s
ok  	l02/ex10	0.106s
```

**Now with the race detector:**

```bash
go test -race -count=1 ./ex10
```

**Sample output** (trimmed — the real report also prints full goroutine creation stacks; addresses and
goroutine IDs differ every run):

```
==================
WARNING: DATA RACE
Read at 0x00c0000b6178 by goroutine 10:
  l02/ex10.MaxParallelRacy.func1()
      /tmp/dsa-t/ex10/parallel.go:25 +0xc4

Previous write at 0x00c0000b6178 by goroutine 11:
  l02/ex10.MaxParallelRacy.func1()
      /tmp/dsa-t/ex10/parallel.go:26 +0xdc
...
==================
--- FAIL: TestMaxParallelRacy (0.00s)
    testing.go:1712: race detected during execution of test
FAIL
FAIL	l02/ex10	0.301s
FAIL
```

Line 25 is the `if v > best` read; line 26 is the `best = v` write. The detector names both.

> Re-verifying this lesson later, the same test **did** fail once — while fourteen packages were
> building at the same time. Twelve isolated runs then passed 12/12. That is the failure mode
> exactly: a real race that shows up under load, on someone else's machine, months later.

**Rust:**

```rust
// Safe Rust cannot express Go's data race: the program below it does not compile.
// So this port keeps the POINT of the example -- "a passing test is not evidence
// for concurrent code" -- and shows what Rust does instead:
//
// 1. What Go lets you write. `best` is shared mutably between threads:
//
//        let mut best = xs[0];
//        thread::scope(|s| {
//            for part in xs.chunks(chunk) {
//                s.spawn(|| for &v in part { if v > best { best = v; } });
//            }
//        });
//
//    rustc rejects it at compile time: error[E0499]: cannot borrow `best` as
//    mutable more than once at a time (plus E0373: the closure borrows the loop
//    variable `part`). Adding `move` silences both, by giving every thread its
//    own COPY of `best`: it compiles, and returns xs[0] every time. That is a
//    different bug, not a race, and not the one in the Go example.
//
// 2. What Rust does let you write: `max_parallel_racy`, the same *logic* bug
//    with atomics. Every access is atomic, so there is no data race (no
//    undefined behaviour, nothing for a race detector to report); but "read,
//    decide, then write" is still not one step, so a stale write can overwrite
//    a larger value. Rust prevents data races; it does not prevent race conditions.
//
// 3. Three ways to get it right, each making the sharing explicit.
//
// Go's `-race` has no stable-Rust counterpart: ThreadSanitizer needs nightly
// (`-Zsanitizer=thread`) and was not run here.
use std::sync::Mutex;
use std::sync::atomic::{AtomicI64, Ordering::Relaxed};
use std::thread;

/// Same shape as Go's MaxParallelRacy: workers reduce into one shared variable
/// with an unsynchronized check-then-act. The race window is a few nanoseconds
/// wide -- two threads can both load `best`, both decide they are larger, and
/// one store is lost.
///
/// It usually returns the right answer anyway -- which is exactly the problem.
fn max_parallel_racy(xs: &[i64], workers: usize) -> i64 {
    let best = AtomicI64::new(xs[0]);
    let shared = &best; // a plain reference is Copy, so each `move` closure gets its own
    let chunk = xs.len().div_ceil(workers);
    thread::scope(|s| {
        for part in xs.chunks(chunk) {
            s.spawn(move || {
                for &v in part {
                    if v > shared.load(Relaxed) {
                        shared.store(v, Relaxed); // <-- stale decision: `best` may have changed since the load
                    }
                }
            });
        }
    });
    best.into_inner()
}

/// The fix from the Go example: each worker reduces its own chunk into a
/// private variable (here, the value its thread returns), and the results are
/// combined once, in one thread. No shared mutable state means no lock and no race.
fn max_parallel(xs: &[i64], workers: usize) -> i64 {
    let chunk = xs.len().div_ceil(workers);
    let partials: Vec<i64> = thread::scope(|s| {
        // `chunks` never yields an empty slice, so `part[0]` is safe.
        let handles: Vec<_> = xs
            .chunks(chunk)
            .map(|part| {
                s.spawn(move || {
                    let mut local = part[0]; // only this thread touches `local`
                    for &v in &part[1..] {
                        if v > local {
                            local = v;
                        }
                    }
                    local
                })
            })
            .collect();
        handles.into_iter().map(|h| h.join().unwrap()).collect()
    });
    partials.into_iter().max().unwrap()
}

/// Shared state, made atomic: `fetch_max` is the read-decide-write as ONE step.
fn max_parallel_atomic(xs: &[i64], workers: usize) -> i64 {
    let best = AtomicI64::new(xs[0]);
    let shared = &best;
    let chunk = xs.len().div_ceil(workers);
    thread::scope(|s| {
        for part in xs.chunks(chunk) {
            s.spawn(move || {
                for &v in part {
                    shared.fetch_max(v, Relaxed);
                }
            });
        }
    });
    best.into_inner()
}

/// Shared state behind a lock. The data lives INSIDE the `Mutex`, so the only
/// way to reach it is through the guard. (`thread::scope` lets threads borrow
/// the mutex, so no `Arc` is needed; with `thread::spawn` you would wrap it in one.)
fn max_parallel_mutex(xs: &[i64], workers: usize) -> i64 {
    let best = Mutex::new(xs[0]);
    let shared = &best;
    let chunk = xs.len().div_ceil(workers);
    thread::scope(|s| {
        for part in xs.chunks(chunk) {
            s.spawn(move || {
                let local = part.iter().copied().max().unwrap();
                let mut guard = shared.lock().unwrap();
                *guard = (*guard).max(local); // one lock per worker, not per element
            });
        }
    });
    best.into_inner().unwrap()
}

type MaxFn = fn(&[i64], usize) -> i64;

const VERSIONS: [(&str, MaxFn); 4] = [
    ("racy (load, then store)", max_parallel_racy),
    ("partials via join", max_parallel),
    ("atomic fetch_max", max_parallel_atomic),
    ("Mutex, one lock/worker", max_parallel_mutex),
];

/// How many of `runs` calls returned something other than the true maximum.
fn wrong_runs(f: MaxFn, xs: &[i64], workers: usize, runs: usize) -> usize {
    let want = xs.iter().copied().max().unwrap();
    (0..runs).filter(|_| f(xs, workers) != want).count()
}

fn main() {
    let input = |n: i64| -> Vec<i64> { (0..n).collect() };
    let small = input(10_000);

    println!("the Go test, ported: input 0..9999, 4 workers, want 9999");
    for (name, f) in VERSIONS {
        println!("  {name:<25}{}", f(&small, 4));
    }

    println!("\nstress: 20000 runs, input 0..9999, 4 workers");
    for (name, f) in VERSIONS {
        println!(
            "  {name:<25}wrong in {} runs",
            wrong_runs(f, &small, 4, 20_000)
        );
    }

    // More shared work means more time inside the race window.
    let big = input(1_000_000);
    println!("\nstress: 300 runs, input 0..999999, 4 workers");
    for (name, f) in VERSIONS {
        println!("  {name:<25}wrong in {} runs", wrong_runs(f, &big, 4, 300));
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O parallel.rs -o parallel && ./parallel
```

**Rust sample output:**

```
the Go test, ported: input 0..9999, 4 workers, want 9999
  racy (load, then store)  9999
  partials via join        9999
  atomic fetch_max         9999
  Mutex, one lock/worker   9999

stress: 20000 runs, input 0..9999, 4 workers
  racy (load, then store)  wrong in 308 runs
  partials via join        wrong in 0 runs
  atomic fetch_max         wrong in 0 runs
  Mutex, one lock/worker   wrong in 0 runs

stress: 300 runs, input 0..999999, 4 workers
  racy (load, then store)  wrong in 53 runs
  partials via join        wrong in 0 runs
  atomic fetch_max         wrong in 0 runs
  Mutex, one lock/worker   wrong in 0 runs
```

> **Go vs Rust:** Go's racy version has no safe-Rust twin: sharing `best` mutably between threads is a compile error (E0499; adding `move` makes it compile by giving each thread a private copy, and it then returns `xs[0]` every time), so a data race cannot be written, and `-race` has no stable counterpart (ThreadSanitizer needs nightly; not run). What can still be written is the same logic bug with atomics (`load` then `store`, no undefined behaviour): it returned the right answer on the single check, as the Go test does, yet was wrong in 1% to 4% of 20,000 runs (1.2% to 3.1% over five runs) and in 4% to 70% of 300 runs on a 1,000,000-element input (11 to 201 runs), while the join-partials, `fetch_max` and `Mutex` versions were wrong in 0 runs every time. The type system stops data races, not race conditions, so a passing test is still not evidence.

**Takeaway:** for concurrent code a passing test is not evidence. `-race` is the evidence. It costs ~10× time and memory, so it's a gate — never a benchmark mode.

---

## 11. Fuzzing writes your regression test

`🟡 medium` · *go test -fuzz*

Back to the buggy binary search from example 5 — but this time nobody tells the test where to look.
A fuzz target takes random bytes, shapes them into a valid input, and compares against an oracle. The
fuzzer explores; coverage guidance steers it toward new code paths.

It found the bug in **0.02 seconds**, automatically **minimized** a 53-byte input down to a
3-element slice, and wrote the failing case to `testdata/` — where it becomes a permanent regression
test that runs on every plain `go test` from now on.

**Steps:**

1. Add seeds with `f.Add` — valid, representative, and *passing*, so the fuzzer has to do the work.
2. In `f.Fuzz`, shape the raw bytes into something that satisfies your function's precondition (here: sorted and deduplicated).
3. Compare against the oracle and fail loudly.
4. Run with `-fuzz` and a time limit, then look at what it wrote.

**`search.go`**

```go
package algo

// Search returns the index of target in the sorted slice xs, or -1.
// It carries the same bug as example 5 -- this time nobody tells the fuzzer
// where to look.
func Search(xs []int, target int) int {
	lo, hi := 0, len(xs)-1
	for lo < hi {
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
```

**`search_test.go`**

```go
package algo

import (
	"slices"
	"testing"
)

func linearSearch(xs []int, target int) int {
	for i, v := range xs {
		if v == target {
			return i
		}
	}
	return -1
}

// sortedSet turns arbitrary fuzzer bytes into a valid input for Search: sorted,
// and deduplicated so that "the" correct index is unambiguous.
//
// Shaping the input like this is most of the work in writing a fuzz target --
// the fuzzer supplies chaos, and you narrow it to your function's precondition.
func sortedSet(data []byte) []int {
	seen := make(map[int]struct{}, len(data))
	xs := make([]int, 0, len(data))
	for _, b := range data {
		v := int(b)
		if _, ok := seen[v]; ok {
			continue
		}
		seen[v] = struct{}{}
		xs = append(xs, v)
	}
	slices.Sort(xs)
	return xs
}

func FuzzSearch(f *testing.F) {
	// Seeds: valid, representative, and all passing. The fuzzer mutates from here.
	f.Add([]byte{2, 4, 6, 8}, byte(4)) // hit on the first probe
	f.Add([]byte{1, 2, 3}, byte(9))    // absent, above the range
	f.Add([]byte{5, 5, 5}, byte(0))    // absent, below the range

	f.Fuzz(func(t *testing.T, data []byte, target byte) {
		xs := sortedSet(data)

		want := linearSearch(xs, int(target))
		got := Search(xs, int(target))
		if got != want {
			t.Fatalf("Search(%v, %d) = %d, oracle says %d", xs, target, got, want)
		}
	})
}
```

**First, confirm the seeds pass** (a plain `go test` runs seeds only, no fuzzing):

```bash
go test ./ex11
```

```
ok  	l02/ex11	0.438s
```

**Now let it hunt:**

```bash
go test -fuzz=FuzzSearch -fuzztime=30s ./ex11
```

**Sample output** (your counterexample and filename will differ):

```
fuzz: elapsed: 0s, gathering baseline coverage: 0/3 completed
fuzz: elapsed: 0s, gathering baseline coverage: 3/3 completed, now fuzzing with 10 workers
fuzz: minimizing 53-byte failing input file
fuzz: elapsed: 0s, minimizing
--- FAIL: FuzzSearch (0.02s)
    --- FAIL: FuzzSearch (0.00s)
        search_test.go:49: Search([4 48 49], 4) = -1, oracle says 0

    Failing input written to testdata/fuzz/FuzzSearch/d8dadd70ed1a164a
    To re-run:
    go test -run=FuzzSearch/d8dadd70ed1a164a
FAIL
```

**Look at what it wrote:**

```bash
cat ex11/testdata/fuzz/FuzzSearch/d8dadd70ed1a164a
```

```
go test fuzz v1
[]byte("0\x041")
byte('\x04')
```

Those bytes are 48, 4, 49 — which `sortedSet` turns into `[4 48 49]`, with target 4. The first element
again: exactly the case example 5's table missed.

**And now the plain test suite reproduces it forever:**

```bash
go test ./ex11
```

```
--- FAIL: FuzzSearch (0.00s)
    --- FAIL: FuzzSearch/d8dadd70ed1a164a (0.00s)
        search_test.go:49: Search([4 48 49], 4) = -1, oracle says 0
FAIL
FAIL	l02/ex11	0.139s
FAIL
```

**Rust:**

```rust
// `go test -fuzz` is coverage-guided: the engine watches which branches each input
// reaches and mutates the inputs that reach new ones. Stable Rust's std has nothing
// like it; the real tools are `cargo-fuzz` (libFuzzer) and `proptest`/`quickcheck`,
// none of which are installed here, so nothing below is their output.
//
// What this file CAN do with std alone: the same four steps of a fuzz target --
// seeds, shape the raw bytes into a valid input, compare against an oracle, shrink
// the failure -- driven by a seeded random generator with NO coverage guidance.
// Because the generator is seeded, the same run finds the same counterexample.
use std::cmp::Ordering;
use std::collections::BTreeSet;

/// Returns the index of `target` in the sorted slice `xs`, or `None`.
/// It carries the same bug as example 5 -- this time nobody tells the fuzzer
/// where to look.
fn search(xs: &[i64], target: i64) -> Option<usize> {
    let (mut lo, mut hi) = (0isize, xs.len() as isize - 1);
    while lo < hi {
        // <-- the bug lives here
        let mid = lo + (hi - lo) / 2;
        let at = mid as usize;
        match xs[at].cmp(&target) {
            Ordering::Equal => return Some(at),
            Ordering::Less => lo = mid + 1,
            Ordering::Greater => hi = mid - 1,
        }
    }
    None
}

fn linear_search(xs: &[i64], target: i64) -> Option<usize> {
    xs.iter().position(|&v| v == target)
}

/// Turns arbitrary fuzzer bytes into a valid input for `search`: sorted, and
/// deduplicated so that "the" correct index is unambiguous. A `BTreeSet` does
/// both in one step.
///
/// Shaping the input like this is most of the work in writing a fuzz target --
/// the fuzzer supplies chaos, and you narrow it to your function's precondition.
fn sorted_set(data: &[u8]) -> Vec<i64> {
    data.iter()
        .map(|&b| i64::from(b))
        .collect::<BTreeSet<_>>()
        .into_iter()
        .collect()
}

/// The fuzz target: one input in, `Err` with a message if the property fails.
fn check(data: &[u8], target: u8) -> Result<(), String> {
    let xs = sorted_set(data);
    let want = linear_search(&xs, i64::from(target));
    let got = search(&xs, i64::from(target));
    if got != want {
        return Err(format!(
            "search({xs:?}, {target}) = {got:?}, oracle says {want:?}"
        ));
    }
    Ok(())
}

/// Seeds: valid, representative, and all passing. The random search starts after them.
const SEEDS: [(&[u8], u8); 3] = [
    (&[2, 4, 6, 8], 4), // hit on the first probe
    (&[1, 2, 3], 9),    // absent, above the range
    (&[5, 5, 5], 0),    // absent, below the range
];

/// Shrinks a failing input by deleting runs of bytes, halving the run length
/// from half the input down to one byte, for as long as it still fails.
/// (The Go engine does this for you and calls it "minimizing".) Deleting one
/// byte at a time is not enough: it shifts every later position, so a 24-byte
/// input can be stuck where every single deletion makes the failure vanish.
fn minimize(mut data: Vec<u8>, target: u8) -> Vec<u8> {
    let mut run = (data.len() / 2).max(1);
    loop {
        let mut shrunk = false;
        let mut start = 0;
        while start < data.len() {
            let end = (start + run).min(data.len());
            let smaller = [&data[..start], &data[end..]].concat();
            if check(&smaller, target).is_err() {
                data = smaller; // keep `start`: new bytes slid into the gap
                shrunk = true;
            } else {
                start += run;
            }
        }
        if run == 1 && !shrunk {
            return data;
        }
        run = (run / 2).max(1);
    }
}

/// splitmix64, seeded: stands in for the engine's input generator.
struct Rng(u64);
impl Rng {
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    /// uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
}

fn report(label: &str, data: &[u8], target: u8) {
    match check(data, target) {
        Ok(()) => println!("  {label}: ok"),
        Err(msg) => println!("  {label}: FAIL  {msg}"),
    }
}

fn main() {
    // First, confirm the seeds pass: a plain test run replays seeds only, no hunting.
    println!("seeds only:");
    for (i, (data, target)) in SEEDS.iter().enumerate() {
        report(&format!("seed {}", i + 1), data, *target);
    }

    // Now let it hunt: random inputs of 0..=64 bytes and a random target byte.
    println!("\nhunting (generator seeded with 1):");
    let mut rng = Rng(1);
    let (iteration, data, target) = (1..)
        .find_map(|n| {
            let len = rng.below(65) as usize;
            let data: Vec<u8> = (0..len).map(|_| rng.below(256) as u8).collect();
            let target = rng.below(256) as u8;
            check(&data, target).is_err().then_some((n, data, target))
        })
        .expect("the search above never stops until it finds a failure");
    println!(
        "  input #{iteration} fails ({} bytes, {} distinct values)",
        data.len(),
        sorted_set(&data).len()
    );

    println!("  minimizing {}-byte failing input", data.len());
    let small = minimize(data, target);
    println!(
        "  minimized to {} byte(s): {:?}, target {target}",
        small.len(),
        small
    );
    report("counterexample", &small, target);

    // Go writes the failing input to testdata/fuzz/ so every later `go test`
    // replays it. Here the equivalent is a line in the corpus you commit.
    println!("\ncorpus entry to commit: ({small:?}, {target})");
    println!("\nplain run with the entry committed:");
    let mut corpus: Vec<(String, Vec<u8>, u8)> = SEEDS
        .iter()
        .enumerate()
        .map(|(i, &(data, target))| (format!("seed {}", i + 1), data.to_vec(), target))
        .collect();
    corpus.push(("regression 1".to_string(), small, target));
    for (label, data, target) in &corpus {
        report(label, data, *target);
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O fuzz.rs -o fuzz && ./fuzz
```

**Rust sample output:**

```
seeds only:
  seed 1: ok
  seed 2: ok
  seed 3: ok

hunting (generator seeded with 1):
  input #41 fails (38 bytes, 35 distinct values)
  minimizing 38-byte failing input
  minimized to 1 byte(s): [236], target 236
  counterexample: FAIL  search([236], 236) = None, oracle says Some(0)

corpus entry to commit: ([236], 236)

plain run with the entry committed:
  seed 1: ok
  seed 2: ok
  seed 3: ok
  regression 1: FAIL  search([236], 236) = None, oracle says Some(0)
```

> **Go vs Rust:** `go test -fuzz` is coverage-guided; std has no such engine, and `cargo-fuzz` or `proptest` are the real tools (neither is installed, neither was run). So this is a seeded random generator with no coverage guidance plus a deletion shrinker, and because the generator is seeded the same counterexample appears on every run. It failed on the 41st random input (38 bytes) and shrank it to one byte, `search([236], 236) = None`; Go's run stopped at the 3-element `[4 48 49]`, but `Search([4], 4)` fails in Go too (checked by running it), so that was not the smallest case. Go's file under `testdata/fuzz/` becomes a line you commit to a corpus; it is replayed in memory here because the program writes no files.

**Takeaway:** fuzzing is an oracle test where you don't have to invent the inputs. **Commit `testdata/fuzz/`** — those files are the regression suite the fuzzer earned for you.

---

> Next tier: [🔴 hard](3-hard.md) — the reusable harness.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
