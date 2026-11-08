# Step 01 — Introduction · 🟡 Medium

Examples **6–10**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

> ⚠️ **This tier measures time and memory, so your numbers will differ from mine.** The outputs below
> were produced on an Apple M-series laptop with Go 1.26.3 — treat them as *sample output*. What must
> match is the **shape**: which column is flat, which grows, and who wins. Examples 8 and 9 are the
> exceptions — allocation counts and the crossover *shape* are stable everywhere.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

> ← Back to the [index](README.md) · Previous tier: [🟢 easy](1-easy.md) · Next tier: [🔴 hard](3-hard.md)

---

## 6. From counting to the clock

`🟡 medium` · *time.Since / three complexity classes*

Now that the counting has told you what to expect, the stopwatch confirms it. The same 200 lookups
over a million elements, answered three ways. Note the guard: all three strategies are checked to
agree before any timing is reported — **a fast wrong answer is not a result**.

**Steps:**

1. Build the same data as a sorted slice and as a set.
2. Time 200 lookups with a linear scan, `slices.BinarySearch`, and a map.
3. Confirm all three found the same number of hits, then print the ratios.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"time"
)

func main() {
	const n = 1_000_000
	const queries = 200

	// The even numbers 0, 2, 4, ... held two ways: a sorted slice and a set.
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i * 2
	}
	set := make(map[int]struct{}, n)
	for _, v := range xs {
		set[v] = struct{}{}
	}

	rng := rand.New(rand.NewPCG(1, 2))
	targets := make([]int, queries)
	for i := range targets {
		targets[i] = rng.IntN(n) * 2
	}

	start := time.Now()
	linHits := 0
	for _, t := range targets {
		for _, v := range xs {
			if v == t {
				linHits++
				break
			}
		}
	}
	linear := time.Since(start)

	start = time.Now()
	binHits := 0
	for _, t := range targets {
		if _, ok := slices.BinarySearch(xs, t); ok {
			binHits++
		}
	}
	binary := time.Since(start)

	start = time.Now()
	mapHits := 0
	for _, t := range targets {
		if _, ok := set[t]; ok {
			mapHits++
		}
	}
	hashed := time.Since(start)

	fmt.Printf("%d lookups over %d elements (all three found %d)\n\n", queries, n, linHits)
	if linHits != binHits || binHits != mapHits {
		panic("the three strategies disagree")
	}

	fmt.Printf("linear scan    O(n)      %10v\n", linear.Round(time.Microsecond))
	fmt.Printf("binary search  O(log n)  %10v   %8.0fx faster\n", binary.Round(time.Microsecond), float64(linear)/float64(binary))
	fmt.Printf("map lookup     O(1)      %10v   %8.0fx faster\n", hashed.Round(time.Microsecond), float64(linear)/float64(hashed))

	fmt.Println()
	fmt.Println("the clock agrees with the counting -- but only because n is big enough")
}
```

**Sample output** (timings vary):

```
200 lookups over 1000000 elements (all three found 200)

linear scan    O(n)        26.713ms
binary search  O(log n)        22µs       1198x faster
map lookup     O(1)            10µs       2596x faster

the clock agrees with the counting -- but only because n is big enough
```

**Rust:**

```rust
use std::collections::HashSet;
use std::hint::black_box;
use std::time::{Duration, Instant};

// Rng: splitmix64, standing in for math/rand (std has no RNG). Seeded, so the
// *targets* are the same on every run; only the timings vary.
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
    /// like rand.IntN(n): uniform-enough in [0, n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
}

// Go prints a time.Duration like "26.713ms" or "22µs". Rust's Debug for
// Duration prints the same shape, so round to the microsecond first.
fn rounded(d: Duration) -> Duration {
    Duration::from_micros(d.as_micros() as u64)
}

fn main() {
    const N: i64 = 1_000_000;
    const QUERIES: usize = 200;

    // The even numbers 0, 2, 4, ... held two ways: a sorted slice and a set.
    let xs: Vec<i64> = (0..N).map(|i| i * 2).collect();
    let set: HashSet<i64> = xs.iter().copied().collect();

    let mut rng = Rng::new(1);
    let targets: Vec<i64> = (0..QUERIES)
        .map(|_| rng.below(N as u64) as i64 * 2)
        .collect();

    // black_box hides the data from the optimiser, so it cannot reason about
    // the values it is searching. Each strategy still has to do the work.
    let (xs, set, targets) = (black_box(&xs), black_box(&set), black_box(&targets));

    let start = Instant::now();
    let mut lin_hits = 0;
    for t in targets {
        if xs.iter().any(|v| v == t) {
            lin_hits += 1;
        }
    }
    let linear = start.elapsed();

    let start = Instant::now();
    let mut bin_hits = 0;
    for t in targets {
        if xs.binary_search(t).is_ok() {
            bin_hits += 1;
        }
    }
    let binary = start.elapsed();

    let start = Instant::now();
    let mut map_hits = 0;
    for t in targets {
        if set.contains(t) {
            map_hits += 1;
        }
    }
    let hashed = start.elapsed();

    println!("{QUERIES} lookups over {N} elements (all three found {lin_hits})\n");
    // A fast wrong answer is not a result: the three must agree before any
    // timing is reported.
    assert!(
        lin_hits == bin_hits && bin_hits == map_hits,
        "the three strategies disagree"
    );

    let (linear_s, binary_s, hashed_s) = (
        linear.as_secs_f64(),
        binary.as_secs_f64(),
        hashed.as_secs_f64(),
    );
    println!("linear scan    O(n)      {:>10?}", rounded(linear));
    println!(
        "binary search  O(log n)  {:>10?}   {:>8.0}x faster",
        rounded(binary),
        linear_s / binary_s
    );
    println!(
        "map lookup     O(1)      {:>10?}   {:>8.0}x faster",
        rounded(hashed),
        linear_s / hashed_s
    );

    println!();
    println!("the clock agrees with the counting: the O(n) scan is orders of magnitude behind");
    println!("binary search and map lookup both finish in microseconds, too short a run to rank");
    println!(
        "them reliably -- one run can flip their order (examples 7 and 9 use a calibrated harness)"
    );
}
```

**Rust sample output:**

```
200 lookups over 1000000 elements (all three found 200)

linear scan    O(n)        25.743ms
binary search  O(log n)         7µs       3432x faster
map lookup     O(1)             7µs       3656x faster

the clock agrees with the counting: the O(n) scan is orders of magnitude behind
binary search and map lookup both finish in microseconds, too short a run to rank
them reliably -- one run can flip their order (examples 7 and 9 use a calibrated harness)
```

> **Go vs Rust:** The O(n) scan took ~26 ms in both languages (Go 26.7 ms; Rust 25.4–26.2 ms in every run) — the same algorithm over the same 8-byte elements. Rust has no `time.Since`; `Instant::now()` / `elapsed()` do the same job, and `{:?}` on a `Duration` prints in Go's `26.713ms` style. Treat only the gap (milliseconds vs microseconds) as a result: across seven runs the binary-search time ranged 6–15 µs and the map time 4–107 µs, so a single 200-lookup run cannot rank those two.

**Complexity:** scan O(n) per query · binary search O(log n), needs sorted input · map O(1) average, needs O(n) build + O(n) space

---

## 7. Benchmarking from a plain program

`🟡 medium` · *testing.Benchmark / b.Loop*

You don't need a `_test.go` file to use Go's benchmark machinery. `testing.Benchmark` runs the same
loop-calibration logic from an ordinary `main`, which is what makes single-file examples like these
possible. Dividing ns/op by n gives the payoff: **ns-per-element stays flat, and that flatness *is*
the definition of O(n)**.

**Steps:**

1. Wrap the work in `func(b *testing.B)` and pass it to `testing.Benchmark`.
2. Use `for b.Loop()` (Go 1.24+) — it calibrates the iteration count and keeps the call alive.
3. Divide `res.NsPerOp()` by n and read the third column.

```go
package main

import (
	"fmt"
	"testing"
)

func sum(xs []int) int {
	total := 0
	for _, v := range xs {
		total += v
	}
	return total
}

func main() {
	fmt.Printf("%9s %14s %18s %14s\n", "n", "ns/op", "ns per element", "B/op")
	for _, n := range []int{100, 1_000, 10_000, 100_000, 1_000_000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i
		}

		// testing.Benchmark runs the same machinery as `go test -bench`,
		// but from an ordinary program: it picks b.N for you and reports ns/op.
		res := testing.Benchmark(func(b *testing.B) {
			b.ReportAllocs()
			for b.Loop() {
				sum(xs)
			}
		})

		fmt.Printf("%9d %14d %18.3f %14d\n", n, res.NsPerOp(), float64(res.NsPerOp())/float64(n), res.AllocedBytesPerOp())
	}

	fmt.Println()
	fmt.Println("ns/op grows 10x with n, ns-per-element stays flat -> that is what O(n) looks like")
}
```

**Sample output** (timings vary):

```
        n          ns/op     ns per element           B/op
      100             28              0.280              0
     1000            234              0.234              0
    10000           2288              0.229              0
   100000          24228              0.242              0
  1000000         245884              0.246              0

ns/op grows 10x with n, ns-per-element stays flat -> that is what O(n) looks like
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// ---- allocation counter: stand-in for runtime.MemStats / testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)
static TOTAL: AtomicUsize = AtomicUsize::new(0); // bytes ever requested

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

/// (allocation events, bytes requested, result) while running `f`.
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

fn sum(xs: &[i64]) -> i64 {
    xs.iter().sum()
}

fn main() {
    println!(
        "{:>9} {:>14} {:>18} {:>14}",
        "n", "ns/op", "ns per element", "B/op"
    );
    let mut per_element = Vec::new();
    for n in [100usize, 1_000, 10_000, 100_000, 1_000_000] {
        let xs: Vec<i64> = (0..n as i64).collect();

        // ns_per_op is the kit's stand-in for testing.Benchmark: it grows the
        // iteration count until the run is long enough to trust, then reports
        // ns/op. black_box keeps the optimiser from hoisting `sum` out of the
        // loop or throwing its result away.
        let ns = ns_per_op(|| {
            black_box(sum(black_box(&xs)));
        });

        // B/op: bytes requested from the heap by one call (deterministic, via
        // the counting allocator -- Go reads it from b.ReportAllocs()).
        let (_, bytes, _) = allocs_of(|| black_box(sum(black_box(&xs))));

        println!("{n:>9} {ns:>14.1} {:>18.3} {bytes:>14}", ns / n as f64);
        per_element.push(ns / n as f64);
    }

    // The flatness is the evidence for O(n), so quantify it instead of
    // asserting it: n grew 10,000x, how far did the per-element cost move?
    let lo = per_element.iter().copied().fold(f64::INFINITY, f64::min);
    let hi = per_element.iter().copied().fold(0.0, f64::max);
    println!();
    println!(
        "n grew 10000x; ns per element moved by only {:.1}x",
        hi / lo
    );
    println!("ns/op grows ~10x per row while ns-per-element stays nearly flat -> O(n)");
}
```

**Rust sample output:**

```
        n          ns/op     ns per element           B/op
      100            4.9              0.049              0
     1000           55.4              0.055              0
    10000          626.2              0.063              0
   100000         7272.8              0.073              0
  1000000        80085.9              0.080              0

n grew 10000x; ns per element moved by only 1.6x
ns/op grows ~10x per row while ns-per-element stays nearly flat -> O(n)
```

> **Go vs Rust:** Std has no `testing.Benchmark`, so the kit's `ns_per_op` does the calibration and `std::hint::black_box` plays the role of `b.Loop` (keep the call and its result alive); `B/op` comes from a counting global allocator and is 0 in both languages. Same shape as Go (~10× per row, nearly flat per element), but ~0.05–0.11 ns/element against the Go sample's 0.23–0.28: `iter().sum()` compiles to a NEON `add.2d` loop that adds 8 elements per iteration (checked with `rustc -O --emit asm`; I did not inspect Go's code). Over several runs the per-element cost drifted up 1.4–2.2× as n grew (plausible mechanism, not measured: the data stops fitting in L1, and the machine was busy).

**Complexity:** O(n) time, O(1) space, **0 allocations** — `sum` never touches the heap

---

## 8. Counting allocations

`🟡 medium` · *testing.AllocsPerRun*

Allocation counts are the best of both worlds: as meaningful as timing, as **reproducible as
counting**. Two functions here are both O(n), both return an identical slice — but one knows the
final size and the other discovers it 25 reallocations later. Preallocation is the cheapest
performance win in Go and it never changes the complexity you'd write down.

**Steps:**

1. Write the same loop twice: `var xs []int` vs `make([]int, 0, n)`.
2. Measure both with `testing.AllocsPerRun` (50 warm runs, returns the average).
3. Assign to a package-level `sink` so the result can't be optimized away.

```go
package main

import (
	"fmt"
	"testing"
)

// Both functions are O(n) and produce an identical slice.
// Only one of them knows how big the answer will be.

func growFromNil(n int) []int {
	var xs []int
	for i := 0; i < n; i++ {
		xs = append(xs, i)
	}
	return xs
}

func growPrealloc(n int) []int {
	xs := make([]int, 0, n)
	for i := 0; i < n; i++ {
		xs = append(xs, i)
	}
	return xs
}

var sink []int

func main() {
	fmt.Printf("%9s %18s %18s\n", "n", "nil-slice allocs", "prealloc allocs")
	for _, n := range []int{10, 100, 1_000, 10_000, 100_000} {
		a := testing.AllocsPerRun(50, func() { sink = growFromNil(n) })
		b := testing.AllocsPerRun(50, func() { sink = growPrealloc(n) })
		fmt.Printf("%9d %18.0f %18.0f\n", n, a, b)
	}

	fmt.Println()
	fmt.Println("same O(n), same output -- one reallocates and copies every time it outgrows cap")
	fmt.Println("allocation counts follow the growth policy, not the clock: this table is stable")
}
```

**Output** (stable — allocation counts don't depend on your CPU):

```
        n   nil-slice allocs    prealloc allocs
       10                  2                  1
      100                  5                  1
     1000                  9                  1
    10000                 16                  1
   100000                 25                  1

same O(n), same output -- one reallocates and copies every time it outgrows cap
allocation counts follow the growth policy, not the clock: this table is stable
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// ---- allocation counter: stand-in for runtime.MemStats / testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events (alloc + realloc)
static TOTAL: AtomicUsize = AtomicUsize::new(0); // bytes ever requested

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

/// (allocation events, bytes requested, result) while running `f`.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, usize, R) {
    let (a0, t0) = (ALLOCS.load(Relaxed), TOTAL.load(Relaxed));
    let r = f();
    (ALLOCS.load(Relaxed) - a0, TOTAL.load(Relaxed) - t0, r)
}

// Both functions are O(n) and produce an identical Vec.
// Only one of them knows how big the answer will be.

fn grow_from_empty(n: i64) -> Vec<i64> {
    let mut xs = Vec::new();
    for i in 0..n {
        xs.push(i);
    }
    xs
}

fn grow_prealloc(n: i64) -> Vec<i64> {
    let mut xs = Vec::with_capacity(n as usize);
    for i in 0..n {
        xs.push(i);
    }
    xs
}

fn main() {
    println!(
        "{:>9} {:>18} {:>18}",
        "n", "empty-Vec allocs", "prealloc allocs"
    );
    for n in [10, 100, 1_000, 10_000, 100_000] {
        // allocs_of counts allocation events (alloc + realloc) made while the
        // closure runs; black_box on the result stops the optimiser deleting
        // the Vec (Go assigns to a package-level `sink` for the same reason).
        let (a, _, v1) = allocs_of(|| black_box(grow_from_empty(n)));
        let (b, _, v2) = allocs_of(|| black_box(grow_prealloc(n)));
        assert_eq!(v1, v2, "the two builders must produce the same Vec");
        println!("{n:>9} {a:>18} {b:>18}");
    }

    // Where do those counts come from? Watch the capacity while pushing.
    // The growth policy is exactly what the allocation counts above measure.
    let mut xs: Vec<i64> = Vec::new();
    let mut caps = vec![xs.capacity()];
    for i in 0..100_000 {
        xs.push(i);
        if xs.capacity() != *caps.last().unwrap() {
            caps.push(xs.capacity());
        }
    }
    println!();
    println!("capacities seen while pushing 100000 elements onto a new Vec:");
    println!(
        "  {caps:?}  (0 = nothing allocated yet) -> {} allocation events",
        caps.len() - 1
    );

    println!();
    println!(
        "same O(n), same output -- one asks the allocator again every time it outgrows capacity"
    );
    println!("allocation counts follow the growth policy, not the clock: this table is stable");
}
```

**Rust output:**

```
        n   empty-Vec allocs    prealloc allocs
       10                  3                  1
      100                  6                  1
     1000                  9                  1
    10000                 13                  1
   100000                 16                  1

capacities seen while pushing 100000 elements onto a new Vec:
  [0, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096, 8192, 16384, 32768, 65536, 131072]  (0 = nothing allocated yet) -> 16 allocation events

same O(n), same output -- one asks the allocator again every time it outgrows capacity
allocation counts follow the growth policy, not the clock: this table is stable
```

> **Go vs Rust:** Deterministic, but not the same numbers as Go: the empty-Vec column is 3, 6, 9, 13, 16 (Go: 2, 5, 9, 16, 25) and the prealloc column is 1 in both. The difference is the growth policy, and the capacity trace in the output shows Rust's: a first block of 4 elements, then strict doubling to 131072 — one allocation event per capacity, so ~log2(n). `testing.AllocsPerRun` has no std twin; the kit's counting global allocator counts `alloc` + `realloc` calls (a `realloc` is counted even if the allocator can extend the block in place). `Vec::with_capacity` is Go's `make([]int, 0, n)`.

**Complexity:** both O(n) time and O(n) space · allocations O(log n) vs **O(1)** — the amortized-append story of [06](../../06-arrays-slices.md)

---

## 9. Finding the crossover point

`🟡 medium` · *Constants vs asymptotics*

The most useful experiment in this lesson. O(1) beats O(n) — *eventually*. Below n=16 on this
machine, hashing the key costs more than scanning the entire slice. Every "which structure should I
use?" question has a crossover, and you can always find it with fifteen lines of benchmark.

**Steps:**

1. For each n, build both a slice and a map holding the same values.
2. Search for the **last** element — the scan's worst case, so the comparison is fair.
3. Print fractional ns/op (`NsPerOp()` truncates to whole nanoseconds, and at n=1 the whole operation costs less than one).

```go
package main

import (
	"fmt"
	"testing"
)

func sliceContains(xs []int, x int) bool {
	for _, v := range xs {
		if v == x {
			return true
		}
	}
	return false
}

func mapContains(m map[int]struct{}, x int) bool {
	_, ok := m[x]
	return ok
}

// nsPerOp keeps the fractional part that BenchmarkResult.NsPerOp truncates away
// -- at n=1 the whole operation costs less than a nanosecond.
func nsPerOp(r testing.BenchmarkResult) float64 {
	return float64(r.T.Nanoseconds()) / float64(r.N)
}

func main() {
	fmt.Printf("%7s %14s %14s   %s\n", "n", "slice ns/op", "map ns/op", "winner")
	crossover := 0

	for n := 1; n <= 1024; n *= 2 {
		xs := make([]int, n)
		m := make(map[int]struct{}, n)
		for i := range xs {
			xs[i] = i
			m[i] = struct{}{}
		}
		target := n - 1 // the last element: the scan's worst case

		sliceRes := testing.Benchmark(func(b *testing.B) {
			for b.Loop() {
				sliceContains(xs, target)
			}
		})
		mapRes := testing.Benchmark(func(b *testing.B) {
			for b.Loop() {
				mapContains(m, target)
			}
		})

		sliceNs, mapNs := nsPerOp(sliceRes), nsPerOp(mapRes)
		winner := "slice"
		if mapNs < sliceNs {
			winner = "map"
			if crossover == 0 {
				crossover = n
			}
		}
		fmt.Printf("%7d %14.2f %14.2f   %s\n", n, sliceNs, mapNs, winner)
	}

	fmt.Println()
	fmt.Printf("O(1) only starts beating O(n) at n >= %d on this machine\n", crossover)
	fmt.Println("below that, the hash costs more than the whole scan -- constants decide small n")
}
```

**Sample output** (the exact crossover moves, the shape doesn't):

```
      n    slice ns/op      map ns/op   winner
      1           1.61           1.69   slice
      2           1.59           1.71   slice
      4           1.61           1.96   slice
      8           2.23           2.95   slice
     16           4.53           3.27   map
     32           8.43           3.27   map
     64          15.84           3.27   map
    128          35.05           3.27   map
    256          66.73           3.28   map
    512         125.83           3.27   map
   1024         242.76           3.34   map

O(1) only starts beating O(n) at n >= 16 on this machine
below that, the hash costs more than the whole scan -- constants decide small n
```

Read the two columns: the slice column **doubles** every row (O(n)); the map column is **flat at ~3.3 ns** (O(1)).

> Note: the ~1.6 ns floor in the first rows is the benchmark harness itself, not the lookup — example 12 proves it.

**Rust:**

```rust
use std::collections::HashSet;
use std::hint::black_box;
use std::time::{Duration, Instant};

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

fn slice_contains(xs: &[i64], x: i64) -> bool {
    for &v in xs {
        if v == x {
            return true;
        }
    }
    false
}

fn set_contains(s: &HashSet<i64>, x: i64) -> bool {
    s.contains(&x)
}

fn main() {
    println!(
        "{:>7} {:>14} {:>14}   winner",
        "n", "slice ns/op", "set ns/op"
    );
    let mut rows = Vec::new();

    let mut n = 1;
    while n <= 1024 {
        let xs: Vec<i64> = (0..n).collect();
        let set: HashSet<i64> = xs.iter().copied().collect();
        let target = n - 1; // the last element: the scan's worst case

        // ns_per_op returns fractional nanoseconds (Go's NsPerOp() truncates
        // to whole ones). black_box on the inputs and the answer keeps both
        // lookups alive and un-hoisted.
        let slice_ns = ns_per_op(|| {
            black_box(slice_contains(black_box(&xs), black_box(target)));
        });
        let set_ns = ns_per_op(|| {
            black_box(set_contains(black_box(&set), black_box(target)));
        });

        let winner = if set_ns < slice_ns { "set" } else { "slice" };
        println!("{n:>7} {slice_ns:>14.2} {set_ns:>14.2}   {winner}");
        rows.push((n, slice_ns, set_ns));
        n *= 2;
    }

    // The crossover is the first n from which the set wins and keeps winning:
    // one noisy early row must not be allowed to move it.
    let crossover = (0..rows.len())
        .find(|&i| rows[i..].iter().all(|&(_, slice, set)| set < slice))
        .map(|i| rows[i].0);

    println!();
    match crossover {
        Some(n) => println!("O(1) only starts beating O(n) at n >= {n} on this machine"),
        None => println!("the set never wins for good in this range on this machine"),
    }

    // Read the two columns, with numbers instead of adjectives.
    let (lo, hi) = rows
        .iter()
        .fold((f64::INFINITY, 0.0f64), |(lo, hi), &(_, _, s)| {
            (lo.min(s), hi.max(s))
        });
    let [.., (_, prev, _), (_, last, _)] = rows[..] else {
        unreachable!("the sweep has more than two rows")
    };
    println!("set column: between {lo:.2} and {hi:.2} ns across a 1024x range of n -> flat, O(1)");
    println!(
        "slice column: x{:.1} on the last doubling of n -> O(n)",
        last / prev
    );
}
```

**Rust sample output:**

```
      n    slice ns/op      set ns/op   winner
      1           0.58           4.80   slice
      2           0.96           4.37   slice
      4           1.27           4.46   slice
      8           2.86           4.37   slice
     16           5.05           4.44   set
     32           9.43           4.39   set
     64          17.63           4.42   set
    128          42.58           4.40   set
    256          86.26           4.45   set
    512         147.97           4.48   set
   1024         274.12           4.53   set

O(1) only starts beating O(n) at n >= 16 on this machine
set column: between 4.37 and 4.80 ns across a 1024x range of n -> flat, O(1)
slice column: x1.9 on the last doubling of n -> O(n)
```

> **Go vs Rust:** Same crossover as Go: the set started winning at n = 16 in all 7 runs, n = 8 went to the slice every time, and n = 16 is close (slice 4.9–5.1 ns vs set 4.3–4.9 ns across runs), so read it as 'somewhere between 8 and 32'. Rust's `HashSet` lookup is flat at ~4.4 ns against Go's ~3.3 ns (plausible mechanism, not measured: `HashSet`'s default SipHash-1-3 hasher is slower than Go's), and the first rows sit at ~0.5 ns instead of Go's ~1.6 ns — that floor is the harness plus `black_box`, which differs per language (example 12 measures it). `ns_per_op` keeps fractional nanoseconds, so no `NsPerOp`-style truncation helper is needed.

**Complexity:** slice O(n) with a tiny constant · map O(1) with a larger one — which is exactly why the crossover exists

---

## 10. What "O(n) space" actually costs

`🟡 medium` · *runtime.MemStats*

Three structures, the same million integers, "O(n) space" for all three — and a **4.7× spread** in
actual bytes. Complexity notation deliberately discards the constant; when you're deciding whether an
index fits in RAM, the constant is the entire question.

**Steps:**

1. Force a GC and read `HeapAlloc` to get a stable "live bytes" baseline.
2. Build each structure in turn, re-measuring after each.
3. Keep everything alive with `runtime.KeepAlive` until after the last measurement, or the GC will collect it mid-experiment.

```go
package main

import (
	"fmt"
	"runtime"
)

// heapAlloc forces a GC and reports live heap bytes -- the closest thing to
// "how much space is this structure actually using".
func heapAlloc() uint64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return ms.HeapAlloc
}

func main() {
	const n = 1_000_000
	base := heapAlloc()

	values := make([]int, n)
	for i := range values {
		values[i] = i
	}
	afterValues := heapAlloc()

	pointers := make([]*int, n)
	for i := range pointers {
		v := i
		pointers[i] = &v
	}
	afterPointers := heapAlloc()

	set := make(map[int]struct{}, n)
	for i := 0; i < n; i++ {
		set[i] = struct{}{}
	}
	afterSet := heapAlloc()

	report := func(label string, delta uint64) {
		fmt.Printf("%-22s %8.1f MB %8.1f bytes/element\n",
			label, float64(delta)/(1<<20), float64(delta)/n)
	}
	report("[]int", afterValues-base)
	report("[]*int", afterPointers-afterValues)
	report("map[int]struct{}", afterSet-afterPointers)

	// Keep everything reachable until after the last measurement.
	runtime.KeepAlive(values)
	runtime.KeepAlive(pointers)
	runtime.KeepAlive(set)

	fmt.Println()
	fmt.Println("all three hold the same 1,000,000 integers -- 'O(n) space' hides a 5x spread")
	fmt.Println("[]*int pays twice: the pointer AND the pointed-to int, each separately allocated")
}
```

**Sample output** (map overhead depends on the Go version's map implementation):

```
[]int                       7.6 MB      8.0 bytes/element
[]*int                     15.3 MB     16.0 bytes/element
map[int]struct{}           36.1 MB     37.8 bytes/element

all three hold the same 1,000,000 integers -- 'O(n) space' hides a 5x spread
[]*int pays twice: the pointer AND the pointed-to int, each separately allocated
```

`[]int` is exactly 8 bytes/element — the int itself, nothing more. `[]*int` is 16: an 8-byte pointer
plus an 8-byte heap object. The map's ~38 buys you the O(1) lookup from example 1: **that's the price
of the hash table**, and now you know what it is.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashSet;
use std::hint::black_box;
use std::process::Command;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// ---- allocation counter: stand-in for runtime.MemStats ----
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

/// bytes currently live on the heap. Go has to force a GC first to get a
/// stable number; Rust frees on scope exit, so the counter is exact at any
/// moment -- no runtime.GC(), no HeapAlloc noise.
fn live_bytes() -> usize {
    LIVE.load(Relaxed)
}

/// bytes the OS has actually given this process (resident set size). The
/// counting allocator only sees what was *asked for*; the allocator under it
/// may round each request up. std has no portable RSS call, so ask `ps`.
fn rss_bytes() -> usize {
    let out = Command::new("ps")
        .args(["-o", "rss=", "-p", &std::process::id().to_string()])
        .output()
        .expect("run ps");
    let kib: usize = String::from_utf8_lossy(&out.stdout)
        .trim()
        .parse()
        .expect("ps prints the RSS in KiB");
    kib * 1024
}

fn main() {
    const N: usize = 1_000_000;
    let (base, rss_base) = (live_bytes(), rss_bytes());

    let values: Vec<i64> = (0..N as i64).collect();
    let (after_values, rss_values) = (live_bytes(), rss_bytes());

    // Vec<Box<i64>> is Go's []*int: an 8-byte pointer per slot, plus a separate
    // 8-byte heap object per element.
    let pointers: Vec<Box<i64>> = (0..N as i64).map(Box::new).collect();
    let (after_pointers, rss_pointers) = (live_bytes(), rss_bytes());

    // with_capacity, like make(map, n): size the table once, up front.
    let mut set: HashSet<i64> = HashSet::with_capacity(N);
    set.extend(0..N as i64);
    let (after_set, rss_set) = (live_bytes(), rss_bytes());

    println!(
        "{:<22} {:>12} {:>13}   {:>13}",
        "", "asked for", "asked/elem", "resident/elem"
    );
    let per_elem = |bytes: usize| bytes as f64 / N as f64;
    let report = |label: &str, asked: usize, resident: usize| {
        println!(
            "{label:<22} {:>8.1} MiB {:>6.1} B/elem   {:>6.1} B/elem",
            asked as f64 / (1 << 20) as f64,
            per_elem(asked),
            per_elem(resident)
        );
    };
    report("Vec<i64>", after_values - base, rss_values - rss_base);
    report(
        "Vec<Box<i64>>",
        after_pointers - after_values,
        rss_pointers - rss_values,
    );
    report(
        "HashSet<i64>",
        after_set - after_pointers,
        rss_set - rss_pointers,
    );

    // No GC means nothing is collected mid-experiment, but the optimiser could
    // still delete structures nobody reads: black_box keeps all three alive
    // (Go's runtime.KeepAlive) until after the last measurement.
    black_box(&values);
    black_box(&pointers);
    black_box(&set);

    let spread = |sizes: [usize; 3]| {
        *sizes.iter().max().unwrap() as f64 / *sizes.iter().min().unwrap() as f64
    };
    let (asked_spread, resident_spread) = (
        spread([
            after_values - base,
            after_pointers - after_values,
            after_set - after_pointers,
        ]),
        spread([
            rss_values - rss_base,
            rss_pointers - rss_values,
            rss_set - rss_pointers,
        ]),
    );
    println!();
    println!("all three hold the same 1,000,000 integers -- 'O(n) space' hides a spread:");
    println!("  {asked_spread:.1}x by bytes asked for, {resident_spread:.1}x by bytes resident");
    println!(
        "Vec<Box<i64>> asked for {:.1} B/elem but holds {:.1}: each 8-byte Box really occupies {:.1} bytes",
        per_elem(after_pointers - after_values),
        per_elem(rss_pointers - rss_values),
        per_elem(rss_pointers - rss_values) - 8.0
    );
}
```

**Rust sample output:**

```
                          asked for    asked/elem   resident/elem
Vec<i64>                    7.6 MiB    8.0 B/elem      8.2 B/elem
Vec<Box<i64>>              15.3 MiB   16.0 B/elem     24.3 B/elem
HashSet<i64>               18.0 MiB   18.9 B/elem     18.9 B/elem

all three hold the same 1,000,000 integers -- 'O(n) space' hides a spread:
  2.4x by bytes asked for, 3.0x by bytes resident
Vec<Box<i64>> asked for 16.0 B/elem but holds 24.3: each 8-byte Box really occupies 16.3 bytes
```

> **Go vs Rust:** No GC means nothing to force: the kit's counting allocator reports live bytes exactly at any instant, so there is no `runtime.GC()` or `ReadMemStats`. Two things differ from Go's 8.0 / 16.0 / 37.8 B/element. First, the 'asked for' column is exact but undercounts: `ps` shows each 8-byte `Box` occupying ~16 bytes (~24 B/element resident for `Vec<Box<i64>>`, here on the macOS system allocator), where Go's output shows 8 bytes per boxed int. Second, a pre-sized `HashSet<i64>` is 18.9 B/element, about half of Go's map — consistent with 2^21 buckets × (8-byte slot + 1 control byte), which I computed but did not verify in the source. So the spread is 2.4× by bytes asked for and 3.0× resident, not Go's ~5×.

**Complexity:** all three O(n) space · the constants are 1×, 2×, and ~4.7× — and the constant is what you have to budget for

---

> Next tier: [🔴 hard](3-hard.md) — where the measurements start lying to you.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
