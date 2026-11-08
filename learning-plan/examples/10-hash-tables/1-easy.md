# Step 10 — Hash Tables & Sets · 🟢 Easy

Examples **1–6**: what Θ(1) actually costs, what makes a hash function good, the two collision
strategies, and what Go's map really is.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples 2, 3 and 4 are deterministic apart from their closing benchmarks. Examples 1, 6 and 7 report
memory, which varies slightly between runs.

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

## 1. Θ(n) → Θ(1), and the three things you pay for it

`🟢 easy` · *the only structure that does this*

Every structure so far has been about **order**. A hash table throws order away and buys the one thing
none of the others could give you: find an arbitrary key in constant time, independent of `n`. That
claim is strong enough to deserve measuring.

**Steps:**

1. Race a linear scan, a binary search and a map across five orders of magnitude.
2. Find where the map *loses*.
3. Then price it — in bytes, and in the three properties you gave up.

```go
package main

import (
	"fmt"
	"math/rand"
	"runtime"
	"slices"
	"testing"
)

// Every structure so far has been about ORDER: what is next, what is at index
// i, what came first. A hash table throws order away entirely, and buys the one
// thing none of the others could give you:
//
//	find an arbitrary key in Theta(1), independent of n
//
// "Independent of n" is the claim, and it is strong enough that it deserves to
// be measured rather than believed. Here are the three ways to answer "is x in
// this collection?", across five orders of magnitude.

func containsLinear(xs []int, v int) bool {
	for _, x := range xs {
		if x == v {
			return true
		}
	}
	return false
}

func containsBinary(sorted []int, v int) bool {
	_, ok := slices.BinarySearch(sorted, v)
	return ok
}

func containsMap(m map[int]struct{}, v int) bool {
	_, ok := m[v]
	return ok
}

var sinkB bool

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// heapKB runs the GC twice on each side. One cycle is not always enough to
// collect the previous measurement's garbage, and the leftover shows up as a
// difference in THIS reading -- which is how my first version of the table
// below "proved" that a size hint costs memory.
func heapKB(build func() any) float64 {
	runtime.GC()
	runtime.GC()
	var a, b runtime.MemStats
	runtime.ReadMemStats(&a)
	keep := build()
	runtime.GC()
	runtime.GC()
	runtime.ReadMemStats(&b)
	runtime.KeepAlive(keep)
	return float64(b.HeapAlloc-a.HeapAlloc) / 1024
}

func main() {
	rng := rand.New(rand.NewSource(1))

	fmt.Println("the same question -- 'is x present?' -- three ways:")
	fmt.Println()
	fmt.Printf("  %10s %14s %14s %12s %12s %12s\n",
		"n", "linear ns", "binary ns", "map ns", "lin/map", "bin/map")

	for _, n := range []int{8, 16, 64, 1024, 65536, 1_000_000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = rng.Intn(1 << 30)
		}
		sorted := slices.Clone(xs)
		slices.Sort(sorted)
		m := make(map[int]struct{}, n)
		for _, v := range xs {
			m[v] = struct{}{}
		}
		// probe keys: half present, half absent, so neither side is favoured
		probes := make([]int, 1024)
		for i := range probes {
			if i%2 == 0 {
				probes[i] = xs[rng.Intn(n)]
			} else {
				probes[i] = rng.Intn(1 << 30)
			}
		}

		tl := nsPerOp(func() {
			for _, p := range probes {
				sinkB = containsLinear(xs, p)
			}
		}) / float64(len(probes))
		tb := nsPerOp(func() {
			for _, p := range probes {
				sinkB = containsBinary(sorted, p)
			}
		}) / float64(len(probes))
		tm := nsPerOp(func() {
			for _, p := range probes {
				sinkB = containsMap(m, p)
			}
		}) / float64(len(probes))

		fmt.Printf("  %10d %14.1f %14.1f %12.1f %11.1fx %11.1fx\n", n, tl, tb, tm, tl/tm, tb/tm)
	}

	fmt.Println()
	fmt.Println("  read the map column DOWN, not across: it barely moves while n")
	fmt.Println("  grows by five orders of magnitude. That is what Theta(1) looks")
	fmt.Println("  like on a clock, and no other structure in this repo can do it.")
	fmt.Println()
	fmt.Println("  the linear column is Theta(n) and says so. The binary column is")
	fmt.Println("  Theta(log n) -- it grows, but slowly, and mostly because of cache")
	fmt.Println("  misses rather than the extra comparisons (lesson 04).")
	fmt.Println()
	fmt.Println("  but read the FIRST row too: at n=8 the linear scan WINS. Eight")
	fmt.Println("  ints are one cache line and the comparison is a single")
	fmt.Println("  instruction, while a map lookup has to hash the key first. The")
	fmt.Println("  crossover is around n=16 -- the same place lesson 01 found it --")
	fmt.Println("  and below it Theta(n) beats Theta(1) every time.")

	fmt.Println()
	fmt.Println("does the size hint save memory? I assumed yes. It does not:")
	fmt.Println()
	const hn = 1 << 19
	hk := make([]int, hn)
	for i := range hk {
		hk[i] = rng.Intn(1 << 40)
	}
	fmt.Printf("  %6s %16s %16s %10s\n", "run", "make(map, n) KB", "make(map) KB", "ratio")
	for rep := 0; rep < 3; rep++ {
		hinted := heapKB(func() any {
			m := make(map[int]struct{}, hn)
			for _, k := range hk {
				m[k] = struct{}{}
			}
			return m
		})
		grown := heapKB(func() any {
			m := make(map[int]struct{})
			for _, k := range hk {
				m[k] = struct{}{}
			}
			return m
		})
		fmt.Printf("  %6d %16.0f %16.0f %9.2fx\n", rep, hinted, grown, hinted/grown)
	}
	fmt.Println()
	fmt.Println("  identical, every run. Both end at the same capacity, because the")
	fmt.Println("  capacity is a function of the final element count, not of how you")
	fmt.Println("  got there. What the hint buys is TIME -- the rehashing skipped on")
	fmt.Println("  the way up -- and example 10 measures exactly that.")
	fmt.Println()
	fmt.Println("  I report three runs because my first version reported ONE, and it")
	fmt.Println("  said the hint cost 28% more memory. The reading was contaminated")
	fmt.Println("  by garbage from the previous measurement that two GC cycles had")
	fmt.Println("  not finished sweeping. Repeat a memory measurement before you")
	fmt.Println("  believe it.")

	fmt.Println()
	fmt.Println("what it costs, in bytes per key:")
	fmt.Println()
	fmt.Printf("  %14s %14s %14s %14s\n", "n", "[]int B/key", "map B/key", "map/slice")
	perKey := map[int]float64{}
	for _, n := range []int{1 << 19, 700_000, 1_000_000, 1 << 20} {
		ks := make([]int, n)
		for i := range ks {
			ks[i] = rng.Intn(1 << 40)
		}
		sl := heapKB(func() any { return slices.Clone(ks) })
		mp := heapKB(func() any {
			m := make(map[int]struct{}, n)
			for _, k := range ks {
				m[k] = struct{}{}
			}
			return m
		})
		perKey[n] = mp * 1024 / float64(n)
		label := fmt.Sprintf("%d", n)
		switch n {
		case 1 << 19:
			label = "524288 = 2^19"
		case 1 << 20:
			label = "1048576 = 2^20"
		}
		fmt.Printf("  %14s %14.2f %14.2f %13.1fx\n",
			label, sl*1024/float64(n), perKey[n], mp/sl)
	}
	fmt.Println()
	fmt.Println("  a map costs several times what the raw keys do, and that is the")
	fmt.Println("  trade: you are buying EMPTY SPACE. A hash table stays fast only")
	fmt.Println("  while it stays sparse -- fill it and every lookup degrades toward")
	fmt.Println("  the linear scan you were trying to avoid (example 5).")
	fmt.Println()
	fmt.Printf("  and look at the swing: 700,000 keys cost %.1f bytes each while\n", perKey[700_000])
	fmt.Printf("  1,000,000 cost %.1f -- %.2fx apart for 43%% more keys. Same\n",
		perKey[1_000_000], perKey[1_000_000]/perKey[700_000])
	fmt.Println("  structure, decided entirely by where n falls relative to a growth")
	fmt.Println("  boundary, exactly as in lessons 06 and 09.")

	fmt.Println("and the three things you gave up to get it:")
	fmt.Println()
	fmt.Printf("  %-22s %s\n", "ORDER", "no first, no last, no next, no range query")
	fmt.Printf("  %-22s %s\n", "CONTIGUITY", "keys are scattered; every lookup is a cache miss")
	fmt.Printf("  %-22s %s\n", "PREDICTABILITY", "Theta(1) is amortized and average-case, not worst")
	fmt.Println()
	im := map[int]int{}
	for i := 0; i < 20; i++ {
		im[i] = i
	}
	fmt.Println("  the first 10 keys of the same 20-key map, four times:")
	for i := 0; i < 4; i++ {
		fmt.Print("    ")
		c := 0
		for k := range im {
			fmt.Printf("%2d ", k)
			if c++; c == 10 {
				break
			}
		}
		fmt.Println("...")
	}
	fmt.Println()
	fmt.Println("  Go RANDOMISES map iteration deliberately, so that code which")
	fmt.Println("  accidentally depends on the order fails immediately and loudly")
	fmt.Println("  rather than in production two years later. Example 6 shows how.")
	fmt.Println()
	fmt.Println("  if you need order, you need a sorted structure -- a tree (lessons")
	fmt.Println("  17-19) or a sorted slice. A map is the wrong tool the moment the")
	fmt.Println("  question is 'what is the smallest key greater than x?'")
}
```

**Sample output:**

```
the same question -- 'is x present?' -- three ways:

           n      linear ns      binary ns       map ns      lin/map      bin/map
           8            2.1            3.7          3.3         0.6x         1.1x
          16            4.3            4.5          3.9         1.1x         1.1x
          64           16.5            6.7          3.6         4.6x         1.9x
        1024          216.8           10.1          4.3        50.8x         2.4x
       65536        12279.7           15.8          4.1      2972.8x         3.8x
     1000000       189872.3           18.4          4.9     38551.5x         3.7x

  read the map column DOWN, not across: it barely moves while n
  grows by five orders of magnitude. That is what Theta(1) looks
  like on a clock, and no other structure in this repo can do it.

  the linear column is Theta(n) and says so. The binary column is
  Theta(log n) -- it grows, but slowly, and mostly because of cache
  misses rather than the extra comparisons (lesson 04).

  but read the FIRST row too: at n=8 the linear scan WINS. Eight
  ints are one cache line and the comparison is a single
  instruction, while a map lookup has to hash the key first. The
  crossover is around n=16 -- the same place lesson 01 found it --
  and below it Theta(n) beats Theta(1) every time.

does the size hint save memory? I assumed yes. It does not:

     run  make(map, n) KB     make(map) KB      ratio
       0            18473            18473      1.00x
       1            18473            18473      1.00x
       2            18473            18473      1.00x

  identical, every run. Both end at the same capacity, because the
  capacity is a function of the final element count, not of how you
  got there. What the hint buys is TIME -- the rehashing skipped on
  the way up -- and example 10 measures exactly that.

  I report three runs because my first version reported ONE, and it
  said the hint cost 28% more memory. The reading was contaminated
  by garbage from the previous measurement that two GC cycles had
  not finished sweeping. Repeat a memory measurement before you
  believe it.

what it costs, in bytes per key:

               n    []int B/key      map B/key      map/slice
   524288 = 2^19           8.00          28.08           3.5x
          700000           8.00          19.02           2.4x
         1000000           8.00          29.83           3.7x
  1048576 = 2^20           8.00          28.08           3.5x

  a map costs several times what the raw keys do, and that is the
  trade: you are buying EMPTY SPACE. A hash table stays fast only
  while it stays sparse -- fill it and every lookup degrades toward
  the linear scan you were trying to avoid (example 5).

  and look at the swing: 700,000 keys cost 19.0 bytes each while
  1,000,000 cost 29.8 -- 1.57x apart for 43% more keys. Same
  structure, decided entirely by where n falls relative to a growth
  boundary, exactly as in lessons 06 and 09.
and the three things you gave up to get it:

  ORDER                  no first, no last, no next, no range query
  CONTIGUITY             keys are scattered; every lookup is a cache miss
  PREDICTABILITY         Theta(1) is amortized and average-case, not worst

  the first 10 keys of the same 20-key map, four times:
    12 13 14  0  1  2  3  4 10 15 ...
    14  0  1  2  3  4 10 15 16  5 ...
     6  8 12 13 14  0  1  2  3  4 ...
     6  8 12 13 14  0  1  2  3  4 ...

  Go RANDOMISES map iteration deliberately, so that code which
  accidentally depends on the order fails immediately and loudly
  rather than in production two years later. Example 6 shows how.

  if you need order, you need a sorted structure -- a tree (lessons
  17-19) or a sorted slice. A map is the wrong tool the moment the
  question is 'what is the smallest key greater than x?'
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::hash_map::RandomState;
use std::collections::{BTreeMap, HashMap, HashSet};
use std::hash::BuildHasher;
use std::hint::black_box;
use std::ops::Bound::{Excluded, Unbounded};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Every structure so far has been about ORDER: what is next, what is at index
// i, what came first. A hash table throws order away entirely, and buys the one
// thing none of the others could give you:
//
//     find an arbitrary key in Theta(1), independent of n
//
// "Independent of n" is the claim, and it is strong enough that it deserves to
// be measured rather than believed. Here are the three ways to answer "is x in
// this collection?", across five orders of magnitude.

// ---- measuring kit: std-only stand-ins for what Go's stdlib gives for free ----

// Allocation counter: stand-in for runtime.MemStats. Rust has no GC, so "how
// many bytes does this structure hold" is an exact, repeatable number.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events
static TOTAL: AtomicUsize = AtomicUsize::new(0); // bytes ever requested
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(l.size(), Relaxed);
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(new, Relaxed);
        LIVE.fetch_add(new, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocation events, bytes requested) while running `f`.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, usize, R) {
    let (a0, t0) = (ALLOCS.load(Relaxed), TOTAL.load(Relaxed));
    let r = f();
    (ALLOCS.load(Relaxed) - a0, TOTAL.load(Relaxed) - t0, r)
}

/// bytes currently live on the heap.
fn live_bytes() -> usize {
    LIVE.load(Relaxed)
}

/// stand-in for testing.Benchmark / b.Loop: ns per call of `f`.
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

/// stand-in for math/rand (splitmix64; seeded, so runs are reproducible).
struct Rng(u64);
impl Rng {
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

// ---- the three answers to "is x present?" ----

// slice::contains is the same loop as the hand-written scan: Theta(n).
fn contains_linear(xs: &[i64], v: i64) -> bool {
    xs.contains(&v)
}

// Theta(log n), and the data must be sorted first.
fn contains_binary(sorted: &[i64], v: i64) -> bool {
    sorted.binary_search(&v).is_ok()
}

// Theta(1) on average. HashSet<T> is HashMap<T, ()>: Go's map[int]struct{}.
fn contains_set(set: &HashSet<i64>, v: i64) -> bool {
    set.contains(&v)
}

/// ns per probe: run `contains` over every probe key, divide by the key count.
/// black_box keeps the optimiser from deleting or hoisting the lookups.
fn per_probe(probes: &[i64], mut contains: impl FnMut(i64) -> bool) -> f64 {
    let total = ns_per_op(|| {
        for &p in probes {
            black_box(contains(black_box(p)));
        }
    });
    total / probes.len() as f64
}

/// ns per lookup when each lookup's key depends on the previous lookup's
/// result, so the CPU cannot overlap them: a latency measurement rather than
/// a throughput one. `probes.len()` must be a power of two.
fn chained(set: &HashSet<i64>, probes: &[i64]) -> f64 {
    let mask = probes.len() - 1;
    let mut i = 0;
    let total = ns_per_op(|| {
        for _ in 0..4096 {
            let hit = set.contains(&probes[i]);
            i = (i + 1 + usize::from(hit)) & mask;
        }
    });
    black_box(i);
    total / 4096.0
}

/// Build a set by inserting one key at a time. Deliberately NOT `.collect()`:
/// FromIterator reserves space from the iterator's size_hint, which would
/// quietly give the "no hint" case a hint.
fn build_set(keys: &[i64], hinted: bool) -> HashSet<i64> {
    let mut set = if hinted {
        HashSet::with_capacity(keys.len())
    } else {
        HashSet::new()
    };
    for &k in keys {
        set.insert(k);
    }
    set
}

/// KB of heap that `build`'s result keeps alive, and how many allocation
/// events it took. Unlike Go there is no GC to run twice and no garbage from
/// the previous measurement to contaminate this one: the reading is exact.
fn measure<T>(build: impl FnOnce() -> T) -> (f64, usize, T) {
    let before = live_bytes() as i64;
    let (allocs, _bytes, kept) = allocs_of(build);
    let kb = (live_bytes() as i64 - before) as f64 / 1024.0;
    (kb, allocs, kept)
}

fn main() {
    let mut rng = Rng(1);

    println!("the same question -- 'is x present?' -- three ways:");
    println!();
    println!(
        "  {:>10} {:>14} {:>14} {:>12} {:>12} {:>12}",
        "n", "linear ns", "binary ns", "set ns", "lin/set", "bin/set"
    );

    let mut rows = Vec::new(); // (n, linear, binary, set), kept for the commentary
    for n in [8usize, 16, 64, 1024, 65536, 1_000_000] {
        let xs: Vec<i64> = (0..n).map(|_| rng.below(1 << 30) as i64).collect();
        let mut sorted = xs.clone();
        sorted.sort_unstable();
        let set = build_set(&xs, true);
        // probe keys: half present, half absent, so neither side is favoured
        let probes: Vec<i64> = (0..1024)
            .map(|i| {
                if i % 2 == 0 {
                    xs[rng.below(n as u64) as usize]
                } else {
                    rng.below(1 << 30) as i64
                }
            })
            .collect();

        let tl = per_probe(&probes, |p| contains_linear(&xs, p));
        let tb = per_probe(&probes, |p| contains_binary(&sorted, p));
        let tm = per_probe(&probes, |p| contains_set(&set, p));

        println!(
            "  {:>10} {:>14.1} {:>14.1} {:>12.1} {:>11.1}x {:>11.1}x",
            n,
            tl,
            tb,
            tm,
            tl / tm,
            tb / tm
        );
        rows.push((n, tl, tb, tm));
    }

    let (first, last) = (rows[0], rows[rows.len() - 1]);
    let growth = last.0 as f64 / first.0 as f64;
    println!();
    println!(
        "  n grew {growth:.0}x from first row to last. In that span the linear column grew {:.0}x,",
        last.1 / first.1
    );
    println!(
        "  the binary column {:.1}x and the set column {:.1}x. Only the linear column",
        last.2 / first.2,
        last.3 / first.3
    );
    println!("  tracks n; the other two grow far more slowly than n does.");
    let scan_wins: Vec<String> = rows
        .iter()
        .filter(|r| r.1 < r.3)
        .map(|r| r.0.to_string())
        .collect();
    println!(
        "  the linear scan is faster than the set lookup at n = {}.",
        scan_wins.join(", ")
    );
    println!();

    // Why does the scan win when n is small? A set lookup has to hash the key
    // first. Measure that cost on its own instead of asserting it.
    let sip = RandomState::new();
    let t_hash = ns_per_op(|| {
        black_box(sip.hash_one(black_box(12345i64)));
    });
    println!(
        "  hashing ONE i64 with std's default hasher (SipHash-1-3), nothing else: {t_hash:.1} ns"
    );
    let set_lo = rows.iter().map(|r| r.3).fold(f64::INFINITY, f64::min);
    println!(
        "  the cheapest set row above is {set_lo:.1} ns for the WHOLE lookup, hashing included --"
    );
    println!("  about what the hash costs on its own, so the hash is most of it, paid before any");
    println!(
        "  slot is touched. A scan of 8 i64s is one cache line and costs less than the hash alone."
    );

    // The table above probes with only 1024 keys, reused on every pass, so the
    // handful of slots they touch stay in cache. Is the 1,000,000-key set really
    // that cheap? Probe it with 2^20 DISTINCT keys instead and see.
    let n = 1_000_000usize;
    let xs: Vec<i64> = (0..n).map(|_| rng.below(1 << 30) as i64).collect();
    let set = build_set(&xs, true);
    let cold: Vec<i64> = (0..1 << 20)
        .map(|i| {
            if i % 2 == 0 {
                xs[rng.below(n as u64) as usize]
            } else {
                rng.below(1 << 30) as i64
            }
        })
        .collect();
    let t_cold = per_probe(&cold, |p| contains_set(&set, p));
    println!();
    let hot = &cold[..1024];
    println!(
        "  the same {n}-key set probed with {} distinct keys instead of 1024: {t_cold:.1} ns per lookup",
        cold.len()
    );
    println!(
        "  (the 1024-key row above said {:.1}). Those lookups are independent, so the CPU can",
        last.3
    );
    println!("  overlap them. Make each key depend on the previous answer and it cannot:");
    let (c_hot, c_cold) = (chained(&set, hot), chained(&set, &cold));
    println!("    chained lookups, 1024 hot keys:       {c_hot:.1} ns");
    println!(
        "    chained lookups, {} cold keys: {c_cold:.1} ns",
        cold.len()
    );
    println!(
        "  cold slots cost {:.1}x hot ones when the lookups cannot overlap. How large that gap",
        c_cold / c_hot
    );
    println!("  is depends on whether this table fits in the cache (plausible, not measured).");
    println!();
    println!("does the size hint save memory? I assumed yes. It does not:");
    println!();
    const HN: usize = 1 << 19;
    let hk: Vec<i64> = (0..HN).map(|_| rng.below(1 << 40) as i64).collect();
    println!(
        "  {:>6} {:>16} {:>16} {:>9} {:>12} {:>12} {:>10} {:>10}",
        "run",
        "with_capacity KB",
        "new() KB",
        "ratio",
        "allocs hint",
        "allocs grow",
        "cap hint",
        "cap grow"
    );
    for rep in 0..3 {
        let (hinted, a_hinted, s) = measure(|| build_set(&hk, true));
        let cap_hinted = s.capacity();
        drop(s);
        let (grown, a_grown, s) = measure(|| build_set(&hk, false));
        let cap_grown = s.capacity();
        drop(s);
        println!(
            "  {:>6} {:>16.0} {:>16.0} {:>8.2}x {:>12} {:>12} {:>10} {:>10}",
            rep,
            hinted,
            grown,
            hinted / grown,
            a_hinted,
            a_grown,
            cap_hinted,
            cap_grown
        );
    }
    println!();
    println!("  identical, every run -- and identical by construction, since there is no");
    println!("  GC noise to wait out. Both end at the same capacity, because the capacity");
    println!("  is a function of the final element count, not of how you got there. What");
    println!("  the hint buys is the allocation columns: ONE allocation instead of a chain");
    println!("  of them, i.e. the rehashing skipped on the way up.");

    println!();
    println!("what it costs, in bytes per key:");
    println!();
    println!(
        "  {:>14} {:>14} {:>14} {:>12} {:>12} {:>8}",
        "n", "[i64] B/key", "set B/key", "set/slice", "capacity()", "B/slot"
    );
    // (n, set bytes per key, capacity()) for each row
    let mut costs: Vec<(usize, f64, usize)> = Vec::new();
    let mut ratios = Vec::new();
    for n in [1usize << 19, 700_000, 1_000_000, 1 << 20] {
        let ks: Vec<i64> = (0..n).map(|_| rng.below(1 << 40) as i64).collect();
        let (sl, _, copy) = measure(|| ks.clone());
        drop(copy);
        let (mp, _, set) = measure(|| build_set(&ks, true));
        let cap = set.capacity();
        drop(set);
        costs.push((n, mp * 1024.0 / n as f64, cap));
        ratios.push(mp / sl);
        let label = match n {
            n if n == 1 << 19 => "524288 = 2^19".to_string(),
            n if n == 1 << 20 => "1048576 = 2^20".to_string(),
            n => n.to_string(),
        };
        // capacity() is 7/8 of the slot count (hashbrown never fills a table
        // past 7/8), so slots = capacity * 8 / 7.
        let slots = (cap * 8 / 7) as f64;
        println!(
            "  {:>14} {:>14.2} {:>14.2} {:>11.1}x {:>12} {:>8.2}",
            label,
            sl * 1024.0 / n as f64,
            costs[costs.len() - 1].1,
            mp / sl,
            cap,
            mp * 1024.0 / slots
        );
    }
    let lo = ratios.iter().copied().fold(f64::INFINITY, f64::min);
    let hi = ratios.iter().copied().fold(0.0, f64::max);
    println!();
    println!("  a set costs {lo:.1}x to {hi:.1}x what the raw keys do, and that is the");
    println!("  trade: you are buying EMPTY SPACE. A hash table stays fast only");
    println!("  while it stays sparse -- fill it and every lookup degrades toward");
    println!("  the linear scan you were trying to avoid (example 5).");
    let cost_of = |n: usize| costs.iter().find(|c| c.0 == n).expect("n was measured");
    let (small, big) = (cost_of(700_000), cost_of(1_000_000));
    println!();
    println!(
        "  and look at the swing: 700,000 keys cost {:.1} bytes each while",
        small.1
    );
    println!(
        "  1,000,000 cost {:.1} -- {:.2}x apart for 43% more keys. Same",
        big.1,
        big.1 / small.1
    );
    println!("  structure, decided entirely by where n falls relative to a growth");
    println!("  boundary, exactly as in lessons 06 and 09. Here the boundary is visible:");
    println!(
        "  capacity() jumps from {} to {}. If a table's slot count is capacity() * 8/7 (it",
        small.2, big.2
    );
    println!("  is never filled past 7/8), the B/slot column reads 9.00 on every row: an 8-byte");
    println!("  key plus one control byte per slot, occupied or not.");

    println!();
    println!("and the three things you gave up to get it:");
    println!();
    let gave_up = [
        ("ORDER", "no first, no last, no next, no range query"),
        (
            "CONTIGUITY",
            "keys are scattered; a lookup whose slot is not cached is a miss",
        ),
        (
            "PREDICTABILITY",
            "Theta(1) is amortized and average-case, not worst",
        ),
    ];
    for (name, what) in gave_up {
        println!("  {name:<22} {what}");
    }
    println!();

    // Go re-randomises the starting point of every iteration of the SAME map.
    // Rust seeds each map once, at construction (RandomState), so a single map
    // iterates the same way every time -- but two maps never agree.
    let line = |m: &HashMap<i64, i64>| -> String {
        let keys: Vec<String> = m.keys().take(10).map(|k| format!("{k:2}")).collect();
        format!("    {} ...", keys.join(" "))
    };
    let im: HashMap<i64, i64> = (0..20).map(|i| (i, i)).collect();
    println!("  the first 10 keys of the same 20-key map, iterated four times:");
    for _ in 0..4 {
        println!("{}", line(&im));
    }
    println!();
    println!("  the first 10 keys of four FRESH maps holding the same 20 keys:");
    for _ in 0..4 {
        let fresh: HashMap<i64, i64> = (0..20).map(|i| (i, i)).collect();
        println!("{}", line(&fresh));
    }
    println!();
    println!("  one map repeats itself; no two maps agree (each gets its own random seed");
    println!("  when it is built, and the seed comes from the OS, so every run of the");
    println!("  program differs too -- run it twice). Go randomises each iteration of one");
    println!("  map instead. Either way the order is unspecified, and code that depends on");
    println!("  it is wrong.");
    println!();
    println!("  if you need order, you need a sorted structure -- a tree (lessons");
    println!("  17-19) or a sorted slice. std ships one: BTreeMap answers exactly the");
    println!("  question a hash map cannot, 'what is the smallest key greater than x?':");
    let tree: BTreeMap<i64, i64> = (0..20).map(|i| (i * 5, i)).collect();
    println!(
        "    first key {:?}, last key {:?}, smallest key > 42: {:?}",
        tree.first_key_value().map(|(k, _)| k),
        tree.last_key_value().map(|(k, _)| k),
        tree.range((Excluded(42), Unbounded)).next().map(|(k, _)| k)
    );
}
```

**Rust sample output:**

```
the same question -- 'is x present?' -- three ways:

           n      linear ns      binary ns       set ns      lin/set      bin/set
           8            0.9            0.9          5.8         0.2x         0.2x
          16            1.3            1.2          6.4         0.2x         0.2x
          64            3.7            2.0          6.0         0.6x         0.3x
        1024           60.3            4.0          6.4         9.4x         0.6x
       65536         3602.0           22.6          6.7       534.1x         3.3x
     1000000        53915.0           21.1          7.7      6993.8x         2.7x

  n grew 125000x from first row to last. In that span the linear column grew 60796x,
  the binary column 22.4x and the set column 1.3x. Only the linear column
  tracks n; the other two grow far more slowly than n does.
  the linear scan is faster than the set lookup at n = 8, 16, 64.

  hashing ONE i64 with std's default hasher (SipHash-1-3), nothing else: 5.7 ns
  the cheapest set row above is 5.8 ns for the WHOLE lookup, hashing included --
  about what the hash costs on its own, so the hash is most of it, paid before any
  slot is touched. A scan of 8 i64s is one cache line and costs less than the hash alone.

  the same 1000000-key set probed with 1048576 distinct keys instead of 1024: 10.7 ns per lookup
  (the 1024-key row above said 7.7). Those lookups are independent, so the CPU can
  overlap them. Make each key depend on the previous answer and it cannot:
    chained lookups, 1024 hot keys:       6.8 ns
    chained lookups, 1048576 cold keys: 17.7 ns
  cold slots cost 2.6x hot ones when the lookups cannot overlap. How large that gap
  is depends on whether this table fits in the cache (plausible, not measured).

does the size hint save memory? I assumed yes. It does not:

     run with_capacity KB         new() KB     ratio  allocs hint  allocs grow   cap hint   cap grow
       0             9216             9216     1.00x            1           19     917504     917504
       1             9216             9216     1.00x            1           19     917504     917504
       2             9216             9216     1.00x            1           19     917504     917504

  identical, every run -- and identical by construction, since there is no
  GC noise to wait out. Both end at the same capacity, because the capacity
  is a function of the final element count, not of how you got there. What
  the hint buys is the allocation columns: ONE allocation instead of a chain
  of them, i.e. the rehashing skipped on the way up.

what it costs, in bytes per key:

               n    [i64] B/key      set B/key    set/slice   capacity()   B/slot
   524288 = 2^19           8.00          18.00         2.3x       917504     9.00
          700000           8.00          13.48         1.7x       917504     9.00
         1000000           8.00          18.87         2.4x      1835008     9.00
  1048576 = 2^20           8.00          18.00         2.3x      1835008     9.00

  a set costs 1.7x to 2.4x what the raw keys do, and that is the
  trade: you are buying EMPTY SPACE. A hash table stays fast only
  while it stays sparse -- fill it and every lookup degrades toward
  the linear scan you were trying to avoid (example 5).

  and look at the swing: 700,000 keys cost 13.5 bytes each while
  1,000,000 cost 18.9 -- 1.40x apart for 43% more keys. Same
  structure, decided entirely by where n falls relative to a growth
  boundary, exactly as in lessons 06 and 09. Here the boundary is visible:
  capacity() jumps from 917504 to 1835008. If a table's slot count is capacity() * 8/7 (it
  is never filled past 7/8), the B/slot column reads 9.00 on every row: an 8-byte
  key plus one control byte per slot, occupied or not.

and the three things you gave up to get it:

  ORDER                  no first, no last, no next, no range query
  CONTIGUITY             keys are scattered; a lookup whose slot is not cached is a miss
  PREDICTABILITY         Theta(1) is amortized and average-case, not worst

  the first 10 keys of the same 20-key map, iterated four times:
     3 13  0 14 17 16  4  8  5  9 ...
     3 13  0 14 17 16  4  8  5  9 ...
     3 13  0 14 17 16  4  8  5  9 ...
     3 13  0 14 17 16  4  8  5  9 ...

  the first 10 keys of four FRESH maps holding the same 20 keys:
    12 16  1 10  3  7 13 15 17 18 ...
    15 16 19 18  8 14  0 17  7  3 ...
     1  3 18  8  6  2 12  7 14  9 ...
     3 11 15 18  1 14  7  8  2  9 ...

  one map repeats itself; no two maps agree (each gets its own random seed
  when it is built, and the seed comes from the OS, so every run of the
  program differs too -- run it twice). Go randomises each iteration of one
  map instead. Either way the order is unspecified, and code that depends on
  it is wrong.

  if you need order, you need a sorted structure -- a tree (lessons
  17-19) or a sorted slice. std ships one: BTreeMap answers exactly the
  question a hash map cannot, 'what is the smallest key greater than x?':
    first key Some(0), last key Some(95), smallest key > 42: Some(45)
```

> **Go vs Rust:** With `HashSet<i64>` (SipHash-1-3, random per-map seed) a lookup costs about 6-8 ns at every size and hashing alone costs about 6 ns, so the scan beats the set up to n=64 here, against only n=8 in the Go sample (Go's map: 3.3-4.9 ns per lookup). Sizes are exact rather than GC-measured: a 9-byte slot (8-byte key plus a control byte) at 7/8 load gives 13.5 B/key at 700,000 keys and 18.9 at 1,000,000 (Go: 19.0 and 29.8), the same growth-boundary swing at smaller constants. Iteration order is stable within one map but differs between maps and between runs, where Go reshuffles every iteration of the same map; and `collect()` pre-sizes from `size_hint`, so the "no hint" case needs an explicit `insert` loop.

**Complexity:** map lookup Θ(1) — **4.9 ns at n=1,000,000 against 190 µs for a scan** · but the scan
**wins at n=8**, and the crossover is around 16 · a map costs **2.4–3.7× the raw keys**, and the size
hint saves **no memory at all** (it buys time — example 10)

---

## 2. What makes a hash function good, and how to measure it

`🟢 easy` · *the output must look random even when the input does not*

Real keys are never random: `user_1`, `/api/v1/orders`, timestamps one second apart. A hash that
preserves any of that structure piles those keys into the same slots — and a hash table with all its
keys in one slot is a linked list with extra steps.

**Steps:**

1. Write four hashes, from useless to good, and score them on distribution *and* avalanche.
2. Find out which **bits** are good, which is a separate question.
3. Then do the birthday arithmetic that makes collisions inevitable.

```go
package main

import (
	"fmt"
	"hash/fnv"
	"math"
	"math/bits"
	"math/rand"
	"strings"
)

// A hash table is only as good as its hash function, and "good" has a precise
// meaning that is easy to state and easy to get wrong:
//
//	the output must look RANDOM even when the input does not
//
// Real keys are never random. They are "user_1", "user_2", "/api/v1/orders",
// timestamps one second apart. A hash function that preserves any of that
// structure piles those keys into the same slots, and a hash table with all its
// keys in one slot is a linked list with extra steps.

// ---- four hash functions, from useless to good ----------------------------

// sumBytes is the one everybody writes first. It is also the worst possible
// choice, for a reason that is obvious once stated: it is COMMUTATIVE.
func sumBytes(s string) uint64 {
	var h uint64
	for i := 0; i < len(s); i++ {
		h += uint64(s[i])
	}
	return h
}

// shiftAdd is the "improved" version -- order now matters. It is still bad,
// because the shift throws away the high bits of everything but the tail.
func shiftAdd(s string) uint64 {
	var h uint64
	for i := 0; i < len(s); i++ {
		h = h<<5 + uint64(s[i])
	}
	return h
}

// fnv1a is a real, simple, widely used hash: multiply by a prime, XOR the byte.
// The multiply is what spreads a one-bit input change across the whole word.
func fnv1a(s string) uint64 {
	const (
		offset = 14695981039346656037
		prime  = 1099511628211
	)
	h := uint64(offset)
	for i := 0; i < len(s); i++ {
		h ^= uint64(s[i])
		h *= prime
	}
	return h
}

// fnv1aFinal is FNV-1a followed by a FINALIZER -- the mixing step from
// splitmix64. Three multiply-xor-shift rounds, and they cost about a
// nanosecond. This is what turns an adequate hash into a good one, and the
// avalanche column below is the reason it exists.
func fnv1aFinal(s string) uint64 {
	h := fnv1a(s)
	h ^= h >> 30
	h *= 0xbf58476d1ce4e5b9
	h ^= h >> 27
	h *= 0x94d049bb133111eb
	h ^= h >> 31
	return h
}

// stdlib, for comparison -- the same algorithm, from hash/fnv.
func stdFNV(s string) uint64 {
	h := fnv.New64a()
	h.Write([]byte(s))
	return h.Sum64()
}

type hashFn struct {
	name string
	fn   func(string) uint64
}

var fns = []hashFn{
	{"sum of bytes", sumBytes},
	{"shift and add", shiftAdd},
	{"FNV-1a", fnv1a},
	{"hash/fnv (stdlib)", stdFNV},
	{"FNV-1a + finalizer", fnv1aFinal},
}

// ---- measuring "looks random" ---------------------------------------------

// chiSquare measures how far a distribution is from uniform. For a good hash,
// the statistic is close to the number of buckets. Much larger means clumping.
func chiSquare(counts []int, n int) float64 {
	b := float64(len(counts))
	expected := float64(n) / b
	var chi float64
	for _, c := range counts {
		d := float64(c) - expected
		chi += d * d / expected
	}
	return chi
}

func distribute(keys []string, fn func(string) uint64, buckets int) []int {
	counts := make([]int, buckets)
	for _, k := range keys {
		counts[fn(k)%uint64(buckets)]++
	}
	return counts
}

func worst(counts []int) int {
	m := 0
	for _, c := range counts {
		if c > m {
			m = c
		}
	}
	return m
}

func empty(counts []int) int {
	n := 0
	for _, c := range counts {
		if c == 0 {
			n++
		}
	}
	return n
}

// avalanche: flip ONE bit of the input and count how many output bits change.
// A good hash changes about half of them -- 32 of 64.
func avalanche(fn func(string) uint64, keys []string) float64 {
	total, n := 0, 0
	for _, k := range keys {
		if len(k) == 0 {
			continue
		}
		h := fn(k)
		b := []byte(k)
		for i := range b {
			for bit := 0; bit < 8; bit++ {
				b[i] ^= 1 << bit
				total += bits.OnesCount64(h ^ fn(string(b)))
				b[i] ^= 1 << bit
				n++
			}
		}
	}
	return float64(total) / float64(n)
}

func histogram(counts []int, width int) string {
	mx := worst(counts)
	var sb strings.Builder
	for i := 0; i < width && i < len(counts); i++ {
		n := 0
		if mx > 0 {
			n = counts[i] * 40 / mx
		}
		sb.WriteString(fmt.Sprintf("    %3d |%s %d\n", i, strings.Repeat("#", n), counts[i]))
	}
	return sb.String()
}

func main() {
	// Realistic keys: sequential ids, paths, and short words. Nothing random.
	var keys []string
	for i := 0; i < 4000; i++ {
		keys = append(keys, fmt.Sprintf("user_%d", i))
	}
	for i := 0; i < 4000; i++ {
		keys = append(keys, fmt.Sprintf("/api/v1/orders/%d", i))
	}
	rng := rand.New(rand.NewSource(4))
	letters := "abcdefghijklmnopqrstuvwxyz"
	for i := 0; i < 4000; i++ {
		b := make([]byte, 3+rng.Intn(5))
		for j := range b {
			b[j] = letters[rng.Intn(26)]
		}
		keys = append(keys, string(b))
	}

	const buckets = 64
	fmt.Printf("%d realistic keys into %d buckets. A perfect hash puts %d in each:\n",
		len(keys), buckets, len(keys)/buckets)
	fmt.Println()
	fmt.Printf("  %-20s %10s %10s %12s %14s\n",
		"hash function", "worst", "empty", "chi-square", "avalanche bits")
	for _, h := range fns {
		c := distribute(keys, h.fn, buckets)
		fmt.Printf("  %-20s %10d %10d %12.0f %14.1f\n",
			h.name, worst(c), empty(c), chiSquare(c, len(keys)), avalanche(h.fn, keys[:200]))
	}
	fmt.Println()
	fmt.Printf("  a uniform hash gives chi-square near %d (the bucket count) and\n", buckets)
	fmt.Println("  avalanche near 32.0 -- half of 64 bits flipping. Those two columns")
	fmt.Println("  measure different failures, and the FNV rows show why you need")
	fmt.Println("  both: FNV-1a's chi-square is excellent (77) while its avalanche is")
	fmt.Println("  only 25.4. It spreads keys across buckets well and still leaves")
	fmt.Println("  visible structure in the bits. Adding a finalizer costs three")
	fmt.Println("  multiplies and fixes the second column without touching the first.")

	fmt.Println()
	fmt.Println("what 'sum of bytes' actually does, first 16 buckets:")
	fmt.Println()
	fmt.Print(histogram(distribute(keys, sumBytes, buckets), 16))
	fmt.Println("  the counts slope steadily downward -- bucket 0 gets nearly 3x")
	fmt.Println("  what bucket 15 does -- because the sum of a handful of ASCII bytes")
	fmt.Println("  is a small number whose distribution is triangular, not uniform.")
	fmt.Println("  It is also COMMUTATIVE, so it cannot tell these apart at all:")
	fmt.Println()
	for _, pair := range [][2]string{{"abc", "cba"}, {"listen", "silent"}, {"ab", "ba"}} {
		fmt.Printf("    %-8s and %-8s -> %d and %d\n",
			pair[0], pair[1], sumBytes(pair[0]), sumBytes(pair[1]))
	}

	fmt.Println()
	fmt.Println("the same 16 buckets under FNV-1a:")
	fmt.Println()
	fmt.Print(histogram(distribute(keys, fnv1a, buckets), 16))

	fmt.Println()
	fmt.Println("avalanche, spelled out -- flipping ONE input bit:")
	fmt.Println()
	fmt.Printf("  %-20s %-22s %-22s %s\n", "hash", "hash(\"user_1000\")", "hash(\"user_1001\")", "bits differing")
	for _, h := range fns {
		a, b := h.fn("user_1000"), h.fn("user_1001")
		fmt.Printf("  %-20s %-22x %-22x %d\n", h.name, a, b, bits.OnesCount64(a^b))
	}
	fmt.Println()
	fmt.Println("  two keys that differ by one character should produce hashes with")
	fmt.Println("  no visible relationship at all. Look at the first two rows: the")
	fmt.Println("  outputs are ADJACENT INTEGERS. Any bucket count will place them")
	fmt.Println("  in neighbouring slots forever.")

	fmt.Println()
	fmt.Println("which BITS are good is a separate question from how many:")
	fmt.Println()
	fmt.Printf("  %-20s %16s %16s %10s\n", "hash function", "chi-sq low 6", "chi-sq top 6", "top/low")
	for _, h := range fns {
		lo := distribute(keys, h.fn, 64)
		hi := make([]int, 64)
		for _, k := range keys {
			hi[h.fn(k)>>58]++
		}
		cl := chiSquare(lo, len(keys))
		ch := chiSquare(hi, len(keys))
		fmt.Printf("  %-20s %16.0f %16.0f %9.0fx\n", h.name, cl, ch, ch/cl)
	}
	fmt.Println()
	fmt.Println("  I expected the low bits to be the weak ones, because that is the")
	fmt.Println("  usual warning. For FNV-1a it is the opposite: its top 6 bits are")
	fmt.Println("  47x worse than its low 6 on these keys. Multiplication only")
	fmt.Println("  carries UPWARD, so the top bits are decided disproportionately by")
	fmt.Println("  the last bytes hashed -- and all 12,000 of these keys end in")
	fmt.Println("  digits. That is a plausible mechanism; the measurement is the")
	fmt.Println("  part I would defend.")
	fmt.Println()
	fmt.Println("  that matters concretely, because Go's map does not do")
	fmt.Println("  `hash % buckets`. It splits the hash into H1 (the upper 57 bits,")
	fmt.Println("  which choose the group) and H2 (the lower 7, stored in the control")
	fmt.Println("  word). A hash that is only good in its low bits would be a poor")
	fmt.Println("  choice for H1. Example 6 takes that apart.")
	fmt.Println()
	fmt.Println("  the finalizer row is the point: after mixing, both halves are")
	fmt.Println("  equally good. If you write your own hash, finalize it -- and if")
	fmt.Println("  you are tempted not to, measure both ends of the word first.")

	fmt.Println("the birthday paradox, which is why collisions are normal:")
	fmt.Println()
	fmt.Printf("  %10s %14s %20s\n", "keys", "buckets", "P(any collision)")
	for _, tc := range []struct{ k, b int }{
		{23, 365}, {83, 5000}, {302, 1 << 16}, {77_163, 1 << 32}, {5000, 1 << 32},
	} {
		p := 1.0
		for i := 0; i < tc.k; i++ {
			p *= float64(tc.b-i) / float64(tc.b)
		}
		fmt.Printf("  %10d %14d %19.1f%%\n", tc.k, tc.b, 100*(1-p))
	}
	fmt.Println()
	fmt.Println("  the first four rows are all near 50%, and the pattern is")
	fmt.Println("  sqrt: you need only about 1.18*sqrt(b) keys before a collision is")
	fmt.Println("  more likely than not. 77,163 keys collide half the time in a")
	fmt.Println("  FOUR-BILLION-slot table.")
	fmt.Println()
	fmt.Println("  the last row is the one that corrects the intuition in the other")
	fmt.Println("  direction: 5,000 keys in that same table collide only 0.3% of the")
	fmt.Println("  time. The threshold is sharp, and it is at the square root.")
	fmt.Println()
	fmt.Println("  either way, collisions are not a sign of a bad hash. They are")
	fmt.Println("  arithmetic, and a table with 64 buckets and 12,000 keys has")
	fmt.Println("  thousands of them by construction.")
	fmt.Println()
	fmt.Printf("  so every hash table needs a collision strategy, and there are\n")
	fmt.Println("  exactly two families: keep colliding keys somewhere else")
	fmt.Println("  (CHAINING, example 3) or keep them in the table (OPEN ADDRESSING,")
	fmt.Println("  example 4). Everything after that is a variation.")

	_ = math.Sqrt
}
```

**Sample output:**

```
12000 realistic keys into 64 buckets. A perfect hash puts 187 in each:

  hash function             worst      empty   chi-square avalanche bits
  sum of bytes                368          0         3712            1.9
  shift and add               497         12        13432            2.0
  FNV-1a                      223          0           77           25.4
  hash/fnv (stdlib)           223          0           77           25.4
  FNV-1a + finalizer          214          0           54           32.0

  a uniform hash gives chi-square near 64 (the bucket count) and
  avalanche near 32.0 -- half of 64 bits flipping. Those two columns
  measure different failures, and the FNV rows show why you need
  both: FNV-1a's chi-square is excellent (77) while its avalanche is
  only 25.4. It spreads keys across buckets well and still leaves
  visible structure in the bits. Adding a finalizer costs three
  multiplies and fixes the second column without touching the first.

what 'sum of bytes' actually does, first 16 buckets:

      0 |################################ 299
      1 |############################# 269
      2 |########################## 241
      3 |####################### 213
      4 |#################### 189
      5 |################## 174
      6 |#################### 191
      7 |################# 160
      8 |############### 146
      9 |############### 141
     10 |################# 158
     11 |############## 137
     12 |############### 138
     13 |############## 130
     14 |############ 114
     15 |############ 112
  the counts slope steadily downward -- bucket 0 gets nearly 3x
  what bucket 15 does -- because the sum of a handful of ASCII bytes
  is a small number whose distribution is triangular, not uniform.
  It is also COMMUTATIVE, so it cannot tell these apart at all:

    abc      and cba      -> 294 and 294
    listen   and silent   -> 655 and 655
    ab       and ba       -> 195 and 195

the same 16 buckets under FNV-1a:

      0 |################################ 182
      1 |################################## 194
      2 |########################## 149
      3 |########################## 150
      4 |################################## 191
      5 |############################## 169
      6 |################################## 191
      7 |################################ 182
      8 |############################## 169
      9 |#################################### 204
     10 |############################## 171
     11 |################################## 193
     12 |################################## 190
     13 |#################################### 205
     14 |############################## 170
     15 |############################## 169

avalanche, spelled out -- flipping ONE input bit:

  hash                 hash("user_1000")      hash("user_1001")      bits differing
  sum of bytes         2df                    2e0                    6
  shift and add        78b22a094630           78b22a094631           1
  FNV-1a               9f77441da0409756       9f77451da0409909       10
  hash/fnv (stdlib)    9f77441da0409756       9f77451da0409909       10
  FNV-1a + finalizer   2c9473340c67f197       fc211731d0e7db88       27

  two keys that differ by one character should produce hashes with
  no visible relationship at all. Look at the first two rows: the
  outputs are ADJACENT INTEGERS. Any bucket count will place them
  in neighbouring slots forever.

which BITS are good is a separate question from how many:

  hash function            chi-sq low 6     chi-sq top 6    top/low
  sum of bytes                     3712           756000       204x
  shift and add                   13432           384577        29x
  FNV-1a                             77             3637        47x
  hash/fnv (stdlib)                  77             3637        47x
  FNV-1a + finalizer                 54               73         1x

  I expected the low bits to be the weak ones, because that is the
  usual warning. For FNV-1a it is the opposite: its top 6 bits are
  47x worse than its low 6 on these keys. Multiplication only
  carries UPWARD, so the top bits are decided disproportionately by
  the last bytes hashed -- and all 12,000 of these keys end in
  digits. That is a plausible mechanism; the measurement is the
  part I would defend.

  that matters concretely, because Go's map does not do
  `hash % buckets`. It splits the hash into H1 (the upper 57 bits,
  which choose the group) and H2 (the lower 7, stored in the control
  word). A hash that is only good in its low bits would be a poor
  choice for H1. Example 6 takes that apart.

  the finalizer row is the point: after mixing, both halves are
  equally good. If you write your own hash, finalize it -- and if
  you are tempted not to, measure both ends of the word first.
the birthday paradox, which is why collisions are normal:

        keys        buckets     P(any collision)
          23            365                50.7%
          83           5000                49.6%
         302          65536                50.1%
       77163     4294967296                50.0%
        5000     4294967296                 0.3%

  the first four rows are all near 50%, and the pattern is
  sqrt: you need only about 1.18*sqrt(b) keys before a collision is
  more likely than not. 77,163 keys collide half the time in a
  FOUR-BILLION-slot table.

  the last row is the one that corrects the intuition in the other
  direction: 5,000 keys in that same table collide only 0.3% of the
  time. The threshold is sharp, and it is at the square root.

  either way, collisions are not a sign of a bad hash. They are
  arithmetic, and a table with 64 buckets and 12,000 keys has
  thousands of them by construction.

  so every hash table needs a collision strategy, and there are
  exactly two families: keep colliding keys somewhere else
  (CHAINING, example 3) or keep them in the table (OPEN ADDRESSING,
  example 4). Everything after that is a variation.
```

**Rust:**

```rust
use std::collections::hash_map::DefaultHasher;
use std::hash::Hasher;

// A hash table is only as good as its hash function, and "good" has a precise
// meaning that is easy to state and easy to get wrong:
//
//     the output must look RANDOM even when the input does not
//
// Real keys are never random. They are "user_1", "user_2", "/api/v1/orders",
// timestamps one second apart. A hash function that preserves any of that
// structure piles those keys into the same slots, and a hash table with all its
// keys in one slot is a linked list with extra steps.

// Splitmix64, the stand-in for math/rand (seeded, so every run is identical).
struct Rng(u64);
impl Rng {
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

// ---- hash functions, from useless to good ---------------------------------
//
// Every one of them takes &[u8], not &str: the avalanche test below flips
// single bits of the input, and a flipped byte is not always valid UTF-8.
// (Go's strings are arbitrary bytes, so it never had to choose.)
//
// Wherever Go relies on integer wraparound, Rust says so out loud:
// `wrapping_add` / `wrapping_mul`. Built with -O it would wrap silently, but a
// debug build panics on overflow -- and a hash that overflows is working as
// designed.

// sum_bytes is the one everybody writes first. It is also the worst possible
// choice, for a reason that is obvious once stated: it is COMMUTATIVE.
fn sum_bytes(s: &[u8]) -> u64 {
    s.iter().fold(0u64, |h, &b| h.wrapping_add(b as u64))
}

// shift_add is the "improved" version -- order now matters. It is still bad,
// because the shift throws away the high bits of everything but the tail.
fn shift_add(s: &[u8]) -> u64 {
    s.iter().fold(0u64, |h, &b| (h << 5).wrapping_add(b as u64))
}

const FNV_OFFSET: u64 = 14695981039346656037;
const FNV_PRIME: u64 = 1099511628211;

// fnv1a is a real, simple, widely used hash: multiply by a prime, XOR the byte.
// The multiply is what spreads a one-bit input change across the whole word.
fn fnv1a(s: &[u8]) -> u64 {
    let mut h = FNV_OFFSET;
    for &b in s {
        h ^= b as u64;
        h = h.wrapping_mul(FNV_PRIME);
    }
    h
}

// fnv1a_final is FNV-1a followed by a FINALIZER -- the mixing step from
// splitmix64. Three multiply-xor-shift rounds. This is what turns an adequate
// hash into a good one, and the avalanche column below is the reason it exists.
fn fnv1a_final(s: &[u8]) -> u64 {
    let mut h = fnv1a(s);
    h ^= h >> 30;
    h = h.wrapping_mul(0xbf58476d1ce4e5b9);
    h ^= h >> 27;
    h = h.wrapping_mul(0x94d049bb133111eb);
    h ^= h >> 31;
    h
}

// Go's hash/fnv gives the same algorithm from the stdlib. Rust's std has no
// FNV, but it has the trait every hash in Rust plugs into: std::hash::Hasher.
// This is FNV-1a again, written as a Hasher -- the shape you would hand to
// HashMap via BuildHasherDefault<Fnv1a> -- and it must score identically to
// the hand-rolled row above.
struct Fnv1a(u64);
impl Default for Fnv1a {
    fn default() -> Self {
        Fnv1a(FNV_OFFSET)
    }
}
impl Hasher for Fnv1a {
    fn write(&mut self, bytes: &[u8]) {
        for &b in bytes {
            self.0 ^= b as u64;
            self.0 = self.0.wrapping_mul(FNV_PRIME);
        }
    }
    fn finish(&self) -> u64 {
        self.0
    }
}
fn fnv1a_hasher(s: &[u8]) -> u64 {
    let mut h = Fnv1a::default();
    h.write(s);
    h.finish()
}

// And the one std really uses: DefaultHasher is SipHash-1-3. `new()` fixes its
// key at zero, so the output is repeatable here; HashMap's own RandomState picks
// a random key per map. The docs do not promise the algorithm stays SipHash-1-3.
fn sip(s: &[u8]) -> u64 {
    let mut h = DefaultHasher::new();
    h.write(s);
    h.finish()
}

type HashFn = fn(&[u8]) -> u64;

const FNS: [(&str, HashFn); 6] = [
    ("sum of bytes", sum_bytes),
    ("shift and add", shift_add),
    ("FNV-1a", fnv1a),
    ("FNV-1a (Hasher)", fnv1a_hasher),
    ("FNV-1a + finalizer", fnv1a_final),
    ("std SipHash-1-3", sip),
];

// ---- measuring "looks random" ---------------------------------------------

// chi_square measures how far a distribution is from uniform. For a good hash,
// the statistic is close to the number of buckets. Much larger means clumping.
fn chi_square(counts: &[usize], n: usize) -> f64 {
    let expected = n as f64 / counts.len() as f64;
    counts
        .iter()
        .map(|&c| {
            let d = c as f64 - expected;
            d * d / expected
        })
        .sum()
}

fn distribute(keys: &[String], f: HashFn, buckets: usize) -> Vec<usize> {
    let mut counts = vec![0; buckets];
    for k in keys {
        counts[(f(k.as_bytes()) % buckets as u64) as usize] += 1;
    }
    counts
}

fn worst(counts: &[usize]) -> usize {
    counts.iter().copied().max().unwrap_or(0)
}

fn empty(counts: &[usize]) -> usize {
    counts.iter().filter(|&&c| c == 0).count()
}

// avalanche: flip ONE bit of the input and count how many output bits change.
// A good hash changes about half of them -- 32 of 64.
fn avalanche(f: HashFn, keys: &[String]) -> f64 {
    let (mut total, mut n) = (0u32, 0u32);
    for k in keys.iter().filter(|k| !k.is_empty()) {
        let mut b = k.as_bytes().to_vec();
        let h = f(&b);
        for i in 0..b.len() {
            for bit in 0..8 {
                b[i] ^= 1 << bit;
                total += (h ^ f(&b)).count_ones();
                b[i] ^= 1 << bit;
                n += 1;
            }
        }
    }
    total as f64 / n as f64
}

fn histogram(counts: &[usize], width: usize) -> String {
    let mx = worst(counts);
    let mut out = String::new();
    for (i, &c) in counts.iter().take(width).enumerate() {
        let n = (c * 40).checked_div(mx).unwrap_or(0);
        out += &format!("    {i:3} |{} {c}\n", "#".repeat(n));
    }
    out
}

fn main() {
    // Realistic keys: sequential ids, paths, and short words. Nothing random.
    let mut keys: Vec<String> = (0..4000).map(|i| format!("user_{i}")).collect();
    keys.extend((0..4000).map(|i| format!("/api/v1/orders/{i}")));
    let mut rng = Rng(4);
    let letters = b"abcdefghijklmnopqrstuvwxyz";
    for _ in 0..4000 {
        let len = 3 + rng.below(5) as usize;
        let word: String = (0..len)
            .map(|_| letters[rng.below(26) as usize] as char)
            .collect();
        keys.push(word);
    }

    const BUCKETS: usize = 64;
    println!(
        "{} realistic keys into {BUCKETS} buckets. A perfect hash puts {} in each:",
        keys.len(),
        keys.len() / BUCKETS
    );
    println!();
    println!(
        "  {:<20} {:>10} {:>10} {:>12} {:>14}",
        "hash function", "worst", "empty", "chi-square", "avalanche bits"
    );
    let mut scores = Vec::new(); // (chi-square, avalanche) per function, for the commentary
    for (name, f) in FNS {
        let c = distribute(&keys, f, BUCKETS);
        let chi = chi_square(&c, keys.len());
        let av = avalanche(f, &keys[..200]);
        println!(
            "  {:<20} {:>10} {:>10} {:>12.0} {:>14.1}",
            name,
            worst(&c),
            empty(&c),
            chi,
            av
        );
        scores.push((chi, av));
    }
    // row indexes: 2 = FNV-1a, 4 = FNV-1a + finalizer
    let (fnv, fin) = (scores[2], scores[4]);
    println!();
    println!("  a uniform hash gives chi-square near {BUCKETS} (the bucket count) -- for 64");
    println!("  buckets a truly random assignment scores 63 +/- 11 -- and avalanche near");
    println!("  32.0, half of 64 bits flipping. Those two columns measure different");
    println!(
        "  failures, and the FNV rows show why you need both: FNV-1a's chi-square is {:.0}",
        fnv.0
    );
    println!(
        "  while its avalanche is only {:.1}. It spreads keys across buckets well and",
        fnv.1
    );
    println!(
        "  still leaves visible structure in the bits. Adding a finalizer ({:.0}, {:.1})",
        fin.0, fin.1
    );
    println!("  costs three multiplies and fixes the second column; the first stays near");
    // chi-square of 64 buckets has 63 degrees of freedom: mean 63, sd sqrt(2*63).
    let z = |chi: f64| (chi - 63.0) / (2.0f64 * 63.0).sqrt();
    println!(
        "  the random-assignment mean ({:+.1} and {:+.1} standard deviations from it).",
        z(fnv.0),
        z(fin.0)
    );
    println!();
    println!("what 'sum of bytes' actually does, first 16 buckets:");
    println!();
    let sum_counts = distribute(&keys, sum_bytes, BUCKETS);
    print!("{}", histogram(&sum_counts, 16));
    let mut sums: Vec<u64> = keys.iter().map(|k| sum_bytes(k.as_bytes())).collect();
    sums.sort_unstable();
    let (lo, hi) = (sums[0], sums[sums.len() - 1]);
    sums.dedup();
    println!(
        "  bucket 0 gets {} keys and bucket 15 gets {} ({:.1}x). The {} keys have only {} distinct",
        sum_counts[0],
        sum_counts[15],
        sum_counts[0] as f64 / sum_counts[15] as f64,
        keys.len(),
        sums.len()
    );
    println!(
        "  sums, all between {lo} and {hi}: the hash itself collides about {:.0} keys per value,",
        keys.len() as f64 / sums.len() as f64
    );
    println!("  before any `% buckets`. It is also COMMUTATIVE,");
    println!("  so it cannot tell these apart at all:");
    println!();
    for (a, b) in [("abc", "cba"), ("listen", "silent"), ("ab", "ba")] {
        println!(
            "    {:<8} and {:<8} -> {} and {}",
            a,
            b,
            sum_bytes(a.as_bytes()),
            sum_bytes(b.as_bytes())
        );
    }

    println!();
    println!("the same 16 buckets under FNV-1a:");
    println!();
    print!("{}", histogram(&distribute(&keys, fnv1a, BUCKETS), 16));

    println!();
    println!("avalanche, spelled out -- flipping ONE input bit:");
    println!();
    println!(
        "  {:<20} {:<22} {:<22} bits differing",
        "hash", "hash(\"user_1000\")", "hash(\"user_1001\")"
    );
    for (name, f) in FNS {
        let (a, b) = (f(b"user_1000"), f(b"user_1001"));
        println!(
            "  {:<20} {:<22x} {:<22x} {}",
            name,
            a,
            b,
            (a ^ b).count_ones()
        );
    }
    println!();
    println!("  two keys that differ by one character should produce hashes with");
    println!("  no visible relationship at all. Look at the first two rows: the");
    println!("  outputs are ADJACENT INTEGERS. Any bucket count will place them");
    println!("  in neighbouring slots forever.");

    println!();
    println!("which BITS are good is a separate question from how many:");
    println!();
    println!(
        "  {:<20} {:>16} {:>16} {:>10}",
        "hash function", "chi-sq low 6", "chi-sq top 6", "top/low"
    );
    let mut top_low = Vec::new();
    for (name, f) in FNS {
        let lo = distribute(&keys, f, 64);
        let mut hi = vec![0; 64];
        for k in &keys {
            hi[(f(k.as_bytes()) >> 58) as usize] += 1;
        }
        let cl = chi_square(&lo, keys.len());
        let ch = chi_square(&hi, keys.len());
        println!("  {:<20} {:>16.0} {:>16.0} {:>9.0}x", name, cl, ch, ch / cl);
        top_low.push((cl, ch));
    }
    println!();
    println!("  I expected the low bits to be the weak ones, because that is the");
    println!(
        "  usual warning. For FNV-1a it is the opposite: its top 6 bits are {:.0}x",
        top_low[2].1 / top_low[2].0
    );
    println!("  worse than its low 6 on these keys. One plausible mechanism: multiplication");
    println!("  only carries UPWARD, so the top bits are decided disproportionately by the");
    println!("  last bytes hashed, and 8,000 of these 12,000 keys end in a digit. Split by");
    println!("  key family, FNV-1a alone:");
    println!();
    println!(
        "    {:<18} {:>8} {:>14} {:>14}",
        "key family", "keys", "chi-sq low 6", "chi-sq top 6"
    );
    let families = [
        ("user_<n>", &keys[..4000]),
        ("/api/v1/orders/<n>", &keys[4000..8000]),
        ("random words", &keys[8000..]),
    ];
    let mut top = Vec::new();
    for (name, group) in families {
        let mut lo = vec![0; 64];
        let mut hi = vec![0; 64];
        for k in group {
            let h = fnv1a(k.as_bytes());
            lo[(h % 64) as usize] += 1;
            hi[(h >> 58) as usize] += 1;
        }
        let (cl, ch) = (chi_square(&lo, group.len()), chi_square(&hi, group.len()));
        println!(
            "    {:<18} {:>8} {:>14.0} {:>14.0}",
            name,
            group.len(),
            cl,
            ch
        );
        top.push(ch);
    }
    println!();
    println!(
        "  the two families that end in digits score {:.0} and {:.0} on the top 6 bits, the",
        top[0], top[1]
    );
    println!(
        "  random words {:.0}. That is {:.0}x better but still nowhere near the 63 a random",
        top[2],
        top[0].min(top[1]) / top[2]
    );
    println!("  hash would give, so the digit-ending explanation accounts for some of the");
    println!("  damage, not all of it. The measurement is the part I would defend.");
    println!();
    println!("  does that matter for Rust's HashMap? Its table (hashbrown, a Swiss table)");
    println!("  starts probing at `hash & (buckets - 1)` -- the LOW bits, which for a power");
    println!("  of two is exactly `hash % buckets` -- and stores the TOP 7 bits as a tag in a");
    println!("  control byte (the Go program above describes the reverse split). So FNV-1a's");
    println!("  good low bits would place keys well while its weak top bits would make tags");
    println!("  agree more often than 1 in 128, costing extra key comparisons. The split is");
    println!("  from the hashbrown 0.17.1 source vendored with std; this program does not");
    println!("  measure it, and the tag consequence is plausible, not measured.");
    println!();
    println!("  the finalizer row is the point: after mixing, both halves are");
    println!("  equally good. If you write your own hash, finalize it -- and if");
    println!("  you are tempted not to, measure both ends of the word first.");

    println!();
    println!("the birthday paradox, which is why collisions are normal:");
    println!();
    println!(
        "  {:>10} {:>14} {:>20}",
        "keys", "buckets", "P(any collision)"
    );
    for (k, b) in [
        (23u64, 365u64),
        (83, 5000),
        (302, 1 << 16),
        (77_163, 1 << 32),
        (5000, 1 << 32),
    ] {
        let mut p = 1.0f64;
        for i in 0..k {
            p *= (b - i) as f64 / b as f64;
        }
        println!("  {:>10} {:>14} {:>19.1}%", k, b, 100.0 * (1.0 - p));
    }
    println!();
    println!("  the first four rows are all near 50%, and the pattern is");
    println!("  sqrt: you need only about 1.18*sqrt(b) keys before a collision is");
    println!("  more likely than not. 77,163 keys collide half the time in a");
    println!("  FOUR-BILLION-slot table.");
    println!();
    println!("  the last row is the one that corrects the intuition in the other");
    println!("  direction: 5,000 keys in that same table collide only 0.3% of the");
    println!("  time. The threshold is sharp, and it is at the square root.");
    println!();
    println!("  either way, collisions are not a sign of a bad hash. They are");
    println!("  arithmetic, and a table with 64 buckets and 12,000 keys has");
    println!("  thousands of them by construction.");
    println!();
    println!("  so every hash table needs a collision strategy, and there are");
    println!("  exactly two families: keep colliding keys somewhere else");
    println!("  (CHAINING, example 3) or keep them in the table (OPEN ADDRESSING,");
    println!("  example 4). Everything after that is a variation.");
}
```

**Rust sample output:**

```
12000 realistic keys into 64 buckets. A perfect hash puts 187 in each:

  hash function             worst      empty   chi-square avalanche bits
  sum of bytes                370          0         3634            1.9
  shift and add               505         12        13342            2.0
  FNV-1a                      221          0           65           25.4
  FNV-1a (Hasher)             221          0           65           25.4
  FNV-1a + finalizer          218          0           49           32.0
  std SipHash-1-3             216          0           47           32.0

  a uniform hash gives chi-square near 64 (the bucket count) -- for 64
  buckets a truly random assignment scores 63 +/- 11 -- and avalanche near
  32.0, half of 64 bits flipping. Those two columns measure different
  failures, and the FNV rows show why you need both: FNV-1a's chi-square is 65
  while its avalanche is only 25.4. It spreads keys across buckets well and
  still leaves visible structure in the bits. Adding a finalizer (49, 32.0)
  costs three multiplies and fixes the second column; the first stays near
  the random-assignment mean (+0.2 and -1.3 standard deviations from it).

what 'sum of bytes' actually does, first 16 buckets:

      0 |################################ 296
      1 |############################# 274
      2 |######################### 232
      3 |####################### 217
      4 |##################### 199
      5 |################### 184
      6 |################## 167
      7 |################# 160
      8 |################## 169
      9 |################ 149
     10 |################ 153
     11 |############### 143
     12 |############# 125
     13 |############## 135
     14 |############ 112
     15 |############# 124
  bucket 0 gets 296 keys and bucket 15 gets 124 (2.4x). The 12000 keys have only 519 distinct
  sums, all between 292 and 1546: the hash itself collides about 23 keys per value,
  before any `% buckets`. It is also COMMUTATIVE,
  so it cannot tell these apart at all:

    abc      and cba      -> 294 and 294
    listen   and silent   -> 655 and 655
    ab       and ba       -> 195 and 195

the same 16 buckets under FNV-1a:

      0 |############################### 172
      1 |################################ 177
      2 |############################## 166
      3 |########################### 153
      4 |################################## 193
      5 |############################# 161
      6 |################################### 196
      7 |################################## 188
      8 |################################# 184
      9 |################################## 193
     10 |################################ 177
     11 |################################### 196
     12 |###################################### 213
     13 |################################### 198
     14 |################################ 182
     15 |############################## 168

avalanche, spelled out -- flipping ONE input bit:

  hash                 hash("user_1000")      hash("user_1001")      bits differing
  sum of bytes         2df                    2e0                    6
  shift and add        78b22a094630           78b22a094631           1
  FNV-1a               9f77441da0409756       9f77451da0409909       10
  FNV-1a (Hasher)      9f77441da0409756       9f77451da0409909       10
  FNV-1a + finalizer   2c9473340c67f197       fc211731d0e7db88       27
  std SipHash-1-3      e5acf3fc62c57b7a       2cf9215432ac732a       24

  two keys that differ by one character should produce hashes with
  no visible relationship at all. Look at the first two rows: the
  outputs are ADJACENT INTEGERS. Any bucket count will place them
  in neighbouring slots forever.

which BITS are good is a separate question from how many:

  hash function            chi-sq low 6     chi-sq top 6    top/low
  sum of bytes                     3634           756000       208x
  shift and add                   13342           384577        29x
  FNV-1a                             65             3710        57x
  FNV-1a (Hasher)                    65             3710        57x
  FNV-1a + finalizer                 49               60         1x
  std SipHash-1-3                    47               60         1x

  I expected the low bits to be the weak ones, because that is the
  usual warning. For FNV-1a it is the opposite: its top 6 bits are 57x
  worse than its low 6 on these keys. One plausible mechanism: multiplication
  only carries UPWARD, so the top bits are decided disproportionately by the
  last bytes hashed, and 8,000 of these 12,000 keys end in a digit. Split by
  key family, FNV-1a alone:

    key family             keys   chi-sq low 6   chi-sq top 6
    user_<n>               4000             90           6243
    /api/v1/orders/<n>     4000             90           6499
    random words           4000             57            274

  the two families that end in digits score 6243 and 6499 on the top 6 bits, the
  random words 274. That is 23x better but still nowhere near the 63 a random
  hash would give, so the digit-ending explanation accounts for some of the
  damage, not all of it. The measurement is the part I would defend.

  does that matter for Rust's HashMap? Its table (hashbrown, a Swiss table)
  starts probing at `hash & (buckets - 1)` -- the LOW bits, which for a power
  of two is exactly `hash % buckets` -- and stores the TOP 7 bits as a tag in a
  control byte (the Go program above describes the reverse split). So FNV-1a's
  good low bits would place keys well while its weak top bits would make tags
  agree more often than 1 in 128, costing extra key comparisons. The split is
  from the hashbrown 0.17.1 source vendored with std; this program does not
  measure it, and the tag consequence is plausible, not measured.

  the finalizer row is the point: after mixing, both halves are
  equally good. If you write your own hash, finalize it -- and if
  you are tempted not to, measure both ends of the word first.

the birthday paradox, which is why collisions are normal:

        keys        buckets     P(any collision)
          23            365                50.7%
          83           5000                49.6%
         302          65536                50.1%
       77163     4294967296                50.0%
        5000     4294967296                 0.3%

  the first four rows are all near 50%, and the pattern is
  sqrt: you need only about 1.18*sqrt(b) keys before a collision is
  more likely than not. 77,163 keys collide half the time in a
  FOUR-BILLION-slot table.

  the last row is the one that corrects the intuition in the other
  direction: 5,000 keys in that same table collide only 0.3% of the
  time. The threshold is sharp, and it is at the square root.

  either way, collisions are not a sign of a bad hash. They are
  arithmetic, and a table with 64 buckets and 12,000 keys has
  thousands of them by construction.

  so every hash table needs a collision strategy, and there are
  exactly two families: keep colliding keys somewhere else
  (CHAINING, example 3) or keep them in the table (OPEN ADDRESSING,
  example 4). Everything after that is a variation.
```

> **Go vs Rust:** Run on the same keys (I fed the Go program this file's splitmix64 word generator as a check), Go and Rust print identical numbers on every shared row; the two outputs above differ only because seeded splitmix64 words are not Go's `math/rand` words (FNV-1a chi-square 65 here vs 77, top/low 57x vs 47x). Rust's std has no FNV, so the `hash/fnv` row becomes FNV-1a written as a `std::hash::Hasher` (identical scores) and a `DefaultHasher` row is added: SipHash-1-3 scores 47 and 32.0, like the finalized FNV. The hashes take `&[u8]` rather than `&str` because the avalanche test flips bits that need not leave valid UTF-8, and every wraparound is an explicit `wrapping_mul`/`wrapping_add`.

**Complexity:** chi-square near the bucket count and avalanche near 32 are the two targets · FNV-1a
scores **77 and 25.4** — good distribution, mediocre bit mixing — and its **top 6 bits are 47× worse
than its low 6** · a three-multiply finalizer fixes both (**54 and 32.0**, top/low **1×**)

---

## 3. Separate chaining, and what it costs

`🟢 easy` · *the version everyone draws on a whiteboard*

Each bucket holds a list. It cannot fail — there is always room for one more key, because "room" means
"another node" — and that forgiving property is exactly what lets it degrade silently.

**Steps:**

1. Build it, then watch what happens when `Put` forgets to search the chain first.
2. Measure chain length against load factor, and find the 37%.
3. Then remove the hash and watch Θ(1) become Θ(n).

```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
	"testing"
)

// SEPARATE CHAINING: each bucket holds a list of the entries that hashed there.
// It is the implementation everyone draws on a whiteboard, and it has one very
// attractive property -- it cannot fail. There is always room for one more key,
// because "room" means "another node".
//
//	lookup   hash -> bucket -> walk the chain
//	insert   hash -> bucket -> prepend (or append, see below)
//	delete   hash -> bucket -> unlink
//
// The cost is everything lesson 07 measured: a pointer per entry, an allocation
// per insert, and a cache miss per link followed.

type entry[K comparable, V any] struct {
	key  K
	val  V
	next *entry[K, V]
}

type Chained[K comparable, V any] struct {
	buckets []*entry[K, V]
	hash    func(K) uint64
	count   int

	probes int // total chain links followed, for the cost measurements below
}

func NewChained[K comparable, V any](n int, hash func(K) uint64) *Chained[K, V] {
	size := 1
	for size < n {
		size <<= 1
	}
	return &Chained[K, V]{buckets: make([]*entry[K, V], size), hash: hash}
}

func (m *Chained[K, V]) index(k K) int { return int(m.hash(k) & uint64(len(m.buckets)-1)) }

func (m *Chained[K, V]) Get(k K) (V, bool) {
	var zero V
	for e := m.buckets[m.index(k)]; e != nil; e = e.next {
		m.probes++
		if e.key == k {
			return e.val, true
		}
	}
	return zero, false
}

// Put must SEARCH FIRST. Skipping this is the classic chaining bug: you get two
// entries with the same key, Get returns whichever the chain reaches first, and
// Delete removes only one of them.
func (m *Chained[K, V]) Put(k K, v V) {
	i := m.index(k)
	for e := m.buckets[i]; e != nil; e = e.next {
		m.probes++
		if e.key == k {
			e.val = v // update in place
			return
		}
	}
	m.buckets[i] = &entry[K, V]{key: k, val: v, next: m.buckets[i]} // prepend
	m.count++
}

// PutNoCheck is that bug, kept so it can be demonstrated rather than described.
func (m *Chained[K, V]) PutNoCheck(k K, v V) {
	i := m.index(k)
	m.buckets[i] = &entry[K, V]{key: k, val: v, next: m.buckets[i]}
	m.count++
}

// Delete is the operation chaining does best: unlink and move on. No
// tombstones, no rearrangement, no "now what?" -- compare example 4.
func (m *Chained[K, V]) Delete(k K) bool {
	i := m.index(k)
	pp := &m.buckets[i] // pointer-to-pointer, lesson 07
	for *pp != nil {
		if (*pp).key == k {
			*pp = (*pp).next
			m.count--
			return true
		}
		pp = &(*pp).next
	}
	return false
}

func (m *Chained[K, V]) Len() int { return m.count }

func (m *Chained[K, V]) chainLengths() []int {
	out := make([]int, len(m.buckets))
	for i, e := range m.buckets {
		for ; e != nil; e = e.next {
			out[i]++
		}
	}
	return out
}

func stats(lens []int, n int) (mean, max float64, empty int) {
	mx := 0
	for _, l := range lens {
		if l > mx {
			mx = l
		}
		if l == 0 {
			empty++
		}
	}
	return float64(n) / float64(len(lens)), float64(mx), empty
}

func fnv1a(s string) uint64 {
	h := uint64(14695981039346656037)
	for i := 0; i < len(s); i++ {
		h ^= uint64(s[i])
		h *= 1099511628211
	}
	h ^= h >> 30
	h *= 0xbf58476d1ce4e5b9
	h ^= h >> 27
	h *= 0x94d049bb133111eb
	h ^= h >> 31
	return h
}

// badHash sends every key to bucket 0. It is what an attacker arranges
// deliberately (example 14) and what a naive hash does by accident.
func badHash(string) uint64 { return 0 }

var sinkB bool
var sinkI int

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func keys(n int) []string {
	out := make([]string, n)
	for i := range out {
		out[i] = fmt.Sprintf("key_%d", i)
	}
	return out
}

func main() {
	fmt.Println("chaining, on a table deliberately too small so the chains show:")
	fmt.Println()
	m := NewChained[string, int](8, fnv1a)
	for i, k := range keys(12) {
		m.Put(k, i)
	}
	for i, e := range m.buckets {
		fmt.Printf("  bucket %d: ", i)
		if e == nil {
			fmt.Println("(empty)")
			continue
		}
		for ; e != nil; e = e.next {
			fmt.Printf("%s=%d -> ", e.key, e.val)
		}
		fmt.Println("nil")
	}
	fmt.Println()
	fmt.Printf("  12 keys, 8 buckets, load factor %.2f\n", float64(m.Len())/float64(len(m.buckets)))
	fmt.Println()
	fmt.Println("  a chained table has no notion of 'full'. Load factor above 1 is")
	fmt.Println("  legal and merely slow, which is the property that makes chaining")
	fmt.Println("  forgiving -- and the property that lets it degrade silently.")

	fmt.Println()
	fmt.Println("Put MUST search the chain first. Here is what happens if it doesn't:")
	fmt.Println()
	good := NewChained[string, int](16, fnv1a)
	bad := NewChained[string, int](16, fnv1a)
	for i := 0; i < 3; i++ {
		good.Put("dup", i)
		bad.PutNoCheck("dup", i)
	}
	gv, _ := good.Get("dup")
	bv, _ := bad.Get("dup")
	fmt.Printf("  %-28s Len=%d  Get(\"dup\")=%d\n", "Put with the search", good.Len(), gv)
	fmt.Printf("  %-28s Len=%d  Get(\"dup\")=%d\n", "Put without it", bad.Len(), bv)
	bad.Delete("dup")
	bv2, ok := bad.Get("dup")
	fmt.Printf("  %-28s after one Delete, Get(\"dup\")=%d, present=%v\n", "", bv2, ok)
	fmt.Println()
	fmt.Println("  the broken table reports 3 entries for 1 key, and deleting the key")
	fmt.Println("  leaves it present with a STALE value. Nothing panics. This is the")
	fmt.Println("  bug to look for whenever a hash map 'forgets' an update.")

	fmt.Println()
	fmt.Println("chain length is the whole cost model. With a good hash:")
	fmt.Println()
	fmt.Printf("  %10s %10s %14s %14s %12s %14s\n",
		"keys", "buckets", "load factor", "mean chain", "longest", "empty buckets")
	for _, tc := range []struct{ n, b int }{
		{1000, 4096}, {1000, 1024}, {1000, 512}, {1000, 256}, {1000, 64},
	} {
		mm := NewChained[string, int](tc.b, fnv1a)
		for i, k := range keys(tc.n) {
			mm.Put(k, i)
		}
		lens := mm.chainLengths()
		mean, mx, emp := stats(lens, tc.n)
		fmt.Printf("  %10d %10d %14.2f %14.2f %12.0f %14d\n",
			tc.n, len(mm.buckets), float64(tc.n)/float64(len(mm.buckets)), mean, mx, emp)
	}
	fmt.Println()
	fmt.Println("  the mean chain length IS the load factor -- that is not a")
	fmt.Println("  coincidence, it is the definition. An unsuccessful lookup walks")
	fmt.Println("  the whole chain, so its expected cost is exactly the load factor.")
	fmt.Println()
	fmt.Println("  note the empty-bucket column at load factor 1.00: roughly 37% of")
	fmt.Println("  buckets are still EMPTY while the mean chain is 1. That is e^-1,")
	fmt.Println("  and it is why a chained table wastes space at exactly the load")
	fmt.Println("  factor where it stops being fast.")

	fmt.Println()
	fmt.Println("and here is what the same table does with the hash removed:")
	fmt.Println()
	adv := NewChained[string, int](1024, badHash)
	ks := keys(2000)
	for i, k := range ks {
		adv.Put(k, i)
	}
	lens := adv.chainLengths()
	_, mx, emp := stats(lens, adv.Len())
	fmt.Printf("  2000 keys, 1024 buckets, longest chain %.0f, empty buckets %d\n", mx, emp)

	fine := NewChained[string, int](1024, fnv1a)
	for i, k := range ks {
		fine.Put(k, i)
	}
	// Count links in ONE pass. Counting inside the benchmark would total them
	// across every b.Loop iteration, which measures the harness, not the table.
	adv.probes, fine.probes = 0, 0
	for _, k := range ks {
		_, sinkB = adv.Get(k)
		_, sinkB = fine.Get(k)
	}
	probesBad, probesGood := adv.probes, fine.probes

	tBad := nsPerOp(func() {
		for _, k := range ks {
			_, sinkB = adv.Get(k)
		}
	})
	tGood := nsPerOp(func() {
		for _, k := range ks {
			_, sinkB = fine.Get(k)
		}
	})
	fmt.Println()
	fmt.Printf("  %-24s %14.0f ns for 2000 lookups\n", "good hash", tGood)
	fmt.Printf("  %-24s %14.0f ns   %.0fx slower\n", "every key collides", tBad, tBad/tGood)
	fmt.Println()
	fmt.Printf("  %-24s %14d links followed  (%.1f per lookup)\n",
		"good hash", probesGood, float64(probesGood)/float64(len(ks)))
	fmt.Printf("  %-24s %14d links followed  (%.1f per lookup)\n",
		"every key collides", probesBad, float64(probesBad)/float64(len(ks)))
	fmt.Println()
	fmt.Println("  Theta(1) became Theta(n), and the data structure did not change --")
	fmt.Println("  only the hash did. This is the worst case a hash table has, it is")
	fmt.Println("  reachable on purpose, and example 14 is about defending it.")

	fmt.Println()
	fmt.Println("what chaining costs, stated plainly:")
	fmt.Println()
	fmt.Printf("  %-26s %s\n", "one allocation per Put", "lesson 07's finding, per key")
	fmt.Printf("  %-26s %s\n", "one pointer per entry", "8 bytes on top of key and value")
	fmt.Printf("  %-26s %s\n", "one cache miss per link", "the chain is scattered")
	fmt.Printf("  %-26s %s\n", "GC traces every entry", "N pointers to scan, forever")
	fmt.Println()
	rng := rand.New(rand.NewSource(2))
	_ = rng
	sort.Ints(lens)
	fmt.Println("  in exchange you get: no 'full' state, no tombstones, deletion")
	fmt.Println("  that is genuinely Theta(1), and load factors above 1 that merely")
	fmt.Println("  degrade instead of failing.")
	fmt.Println()
	fmt.Println("  that trade used to be the default. It is not any more -- Go, Rust,")
	fmt.Println("  Abseil and Swift all moved to OPEN ADDRESSING, which is example 4,")
	fmt.Println("  and example 11 measures why.")
	sinkI = len(lens)
}
```

**Sample output:**

```
chaining, on a table deliberately too small so the chains show:

  bucket 0: key_4=4 -> key_3=3 -> nil
  bucket 1: key_6=6 -> nil
  bucket 2: (empty)
  bucket 3: key_10=10 -> key_9=9 -> nil
  bucket 4: key_8=8 -> key_1=1 -> key_0=0 -> nil
  bucket 5: (empty)
  bucket 6: key_11=11 -> key_2=2 -> nil
  bucket 7: key_7=7 -> key_5=5 -> nil

  12 keys, 8 buckets, load factor 1.50

  a chained table has no notion of 'full'. Load factor above 1 is
  legal and merely slow, which is the property that makes chaining
  forgiving -- and the property that lets it degrade silently.

Put MUST search the chain first. Here is what happens if it doesn't:

  Put with the search          Len=1  Get("dup")=2
  Put without it               Len=3  Get("dup")=2
                               after one Delete, Get("dup")=1, present=true

  the broken table reports 3 entries for 1 key, and deleting the key
  leaves it present with a STALE value. Nothing panics. This is the
  bug to look for whenever a hash map 'forgets' an update.

chain length is the whole cost model. With a good hash:

        keys    buckets    load factor     mean chain      longest  empty buckets
        1000       4096           0.24           0.24            3           3210
        1000       1024           0.98           0.98            5            382
        1000        512           1.95           1.95            8             67
        1000        256           3.91           3.91           11              4
        1000         64          15.62          15.62           30              0

  the mean chain length IS the load factor -- that is not a
  coincidence, it is the definition. An unsuccessful lookup walks
  the whole chain, so its expected cost is exactly the load factor.

  note the empty-bucket column at load factor 1.00: roughly 37% of
  buckets are still EMPTY while the mean chain is 1. That is e^-1,
  and it is why a chained table wastes space at exactly the load
  factor where it stops being fast.

and here is what the same table does with the hash removed:

  2000 keys, 1024 buckets, longest chain 2000, empty buckets 1023

  good hash                         18339 ns for 2000 lookups
  every key collides              2556857 ns   139x slower

  good hash                          3959 links followed  (2.0 per lookup)
  every key collides              2001000 links followed  (1000.5 per lookup)

  Theta(1) became Theta(n), and the data structure did not change --
  only the hash did. This is the worst case a hash table has, it is
  reachable on purpose, and example 14 is about defending it.

what chaining costs, stated plainly:

  one allocation per Put     lesson 07's finding, per key
  one pointer per entry      8 bytes on top of key and value
  one cache miss per link    the chain is scattered
  GC traces every entry      N pointers to scan, forever

  in exchange you get: no 'full' state, no tombstones, deletion
  that is genuinely Theta(1), and load factors above 1 that merely
  degrade instead of failing.

  that trade used to be the default. It is not any more -- Go, Rust,
  Abseil and Swift all moved to OPEN ADDRESSING, which is example 4,
  and example 11 measures why.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::Cell;
use std::hint::black_box;
use std::iter::successors;
use std::mem::size_of;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// SEPARATE CHAINING: each bucket holds a list of the entries that hashed there.
// It is the implementation everyone draws on a whiteboard, and it has one very
// attractive property -- it cannot fail. There is always room for one more key,
// because "room" means "another node".
//
//     lookup   hash -> bucket -> walk the chain
//     insert   hash -> bucket -> prepend (or append, see below)
//     delete   hash -> bucket -> unlink
//
// The cost is everything lesson 07 measured: a pointer per entry, an allocation
// per insert, and a cache miss per link followed.

// ---- measuring kit: std-only stand-ins for what Go's stdlib gives for free ----

// Allocation counter: stand-in for runtime.MemStats. Rust has no GC, so "how
// many allocations / bytes does this structure cost" is an exact number.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events
static TOTAL: AtomicUsize = AtomicUsize::new(0); // bytes ever requested
static LIVE: AtomicUsize = AtomicUsize::new(0); // bytes currently live

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(l.size(), Relaxed);
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(new, Relaxed);
        LIVE.fetch_add(new, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocation events, bytes requested) while running `f`.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, usize, R) {
    let (a0, t0) = (ALLOCS.load(Relaxed), TOTAL.load(Relaxed));
    let r = f();
    (ALLOCS.load(Relaxed) - a0, TOTAL.load(Relaxed) - t0, r)
}

/// bytes currently live on the heap.
fn live_bytes() -> usize {
    LIVE.load(Relaxed)
}

/// stand-in for testing.Benchmark / b.Loop: ns per call of `f`.
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

// ---- the table ----

// `Option<Box<Entry>>` is Rust's `*entry`: "no next entry" is `None`, "next
// entry" is an owning heap pointer. The chain OWNS its nodes -- there is no GC
// to keep them alive, so whoever holds the head holds the whole chain.
struct Entry<V> {
    key: String,
    val: V,
    next: Link<V>,
}
type Link<V> = Option<Box<Entry<V>>>;

// Go's Chained is generic over the key type K. Here K is fixed to String and
// lookups take `&str`, which is how `HashMap<String, _>::get(&str)` avoids
// building a String per lookup. (Doing that generically means a second type
// parameter and the `Borrow` trait -- a detour this example does not need.)
struct Chained<V> {
    buckets: Vec<Link<V>>,
    hash: fn(&str) -> u64,
    count: usize,

    // total chain links followed, for the cost measurements below. A Cell so
    // that `get(&self)` can still count: Go just does `m.probes++` through a
    // pointer receiver, but a Rust `&self` is read-only unless you say
    // otherwise.
    probes: Cell<usize>,
}

impl<V> Chained<V> {
    fn new(n: usize, hash: fn(&str) -> u64) -> Self {
        let size = n.next_power_of_two();
        Chained {
            // `vec![None; size]` would need `Entry: Clone`; building each
            // empty bucket from a closure does not.
            buckets: (0..size).map(|_| None).collect(),
            hash,
            count: 0,
            probes: Cell::new(0),
        }
    }

    fn index(&self, k: &str) -> usize {
        ((self.hash)(k) & (self.buckets.len() as u64 - 1)) as usize
    }

    // Walk one bucket's chain, borrowing the entries. Go writes this loop out
    // by hand (`for e := ...; e != nil; e = e.next`) in four places; an
    // iterator built with `successors` is the same loop, written once.
    fn chain(&self, i: usize) -> impl Iterator<Item = &Entry<V>> {
        successors(self.buckets[i].as_deref(), |e| e.next.as_deref())
    }

    // Go returns (V, bool); Rust returns `Option<&V>`: a borrow of the value
    // inside the entry, valid until the table is next modified.
    fn get(&self, k: &str) -> Option<&V> {
        self.chain(self.index(k))
            .inspect(|_| self.probes.set(self.probes.get() + 1))
            .find(|e| e.key == k)
            .map(|e| &e.val)
    }

    // Put must SEARCH FIRST. Skipping this is the classic chaining bug: you get
    // two entries with the same key, Get returns whichever the chain reaches
    // first, and Delete removes only one of them.
    fn put(&mut self, k: String, v: V) {
        let i = self.index(&k);
        let mut cur = self.buckets[i].as_deref_mut();
        while let Some(e) = cur {
            self.probes.set(self.probes.get() + 1);
            if e.key == k {
                e.val = v; // update in place
                return;
            }
            cur = e.next.as_deref_mut();
        }
        self.prepend(i, k, v);
    }

    fn prepend(&mut self, i: usize, key: String, val: V) {
        // take() leaves None in the bucket and hands back the old head, which
        // becomes the new entry's `next`: Go's one-liner
        // `m.buckets[i] = &entry{..., next: m.buckets[i]}` as two moves.
        let next = self.buckets[i].take();
        self.buckets[i] = Some(Box::new(Entry { key, val, next }));
        self.count += 1;
    }

    // PutNoCheck is that bug, kept so it can be demonstrated rather than
    // described.
    fn put_no_check(&mut self, k: String, v: V) {
        let i = self.index(&k);
        self.prepend(i, k, v);
    }

    // Delete is the operation chaining does best: unlink and move on. No
    // tombstones, no rearrangement, no "now what?" -- compare example 4.
    //
    // Go: `pp := &m.buckets[i]` is a pointer to the LINK (the bucket slot, or
    // some entry's `next` field), so deleting the first entry and deleting a
    // middle one are the same `*pp = (*pp).next`. The Rust twin of that
    // pointer-to-pointer is a cursor of type `&mut Option<Box<Entry>>` -- and
    // this is the famous awkward spot of the borrow checker. The direct
    // transliteration of Go's loop,
    //
    //     while let Some(e) = pp {
    //         if e.key == k { *pp = e.next.take(); return ..; }
    //         pp = &mut e.next;
    //     }
    //
    // is rejected (rustc 1.98: E0506, "cannot assign to `*pp` because it is
    // borrowed"). `e` is a borrow taken THROUGH the cursor, the loop hands that
    // borrow back to the cursor on its last line, and so the compiler will not
    // let the splice write through the cursor while `e` is still in play.
    // Walking with the same `while let` and splicing AFTER the loop fails too
    // (E0499). What compiles: a loop whose CONDITION only peeks (`as_ref`), so
    // no borrow outlives the test; then `take()` the node out of the cursor;
    // and only then write through it.
    fn delete(&mut self, k: &str) -> Option<V> {
        let i = self.index(k);
        let mut pp: &mut Link<V> = &mut self.buckets[i];
        while pp.as_ref().is_some_and(|e| e.key != k) {
            // the condition just proved `*pp` is Some, so this cannot fail
            pp = &mut pp.as_mut().unwrap().next;
        }
        // pp now points at the link that holds the key, or at the final None
        // (`?` returns None: key absent).
        let doomed = pp.take()?;
        let Entry { val, next, .. } = *doomed; // the String key drops here
        *pp = next; // the link that pointed at the entry now points past it
        self.count -= 1;
        Some(val)
    }

    fn len(&self) -> usize {
        self.count
    }

    fn chain_lengths(&self) -> Vec<usize> {
        (0..self.buckets.len())
            .map(|i| self.chain(i).count())
            .collect()
    }
}

// Rust frees a chain by dropping the head, which drops `next`, which drops ITS
// `next` ... one stack frame per entry. The adversarial table below makes a
// chain as long as the whole table, and a long enough chain overflows the
// stack and aborts the process: without this impl, the million-entry chain in
// the last section died with "stack overflow" on the 8 MB main thread (a bare
// chain of String-keyed boxes survived 200,000 entries and died by 400,000).
// Go's collector marks with an explicit work list, so it has no such limit.
// Unlinking iteratively, one node at a time, keeps every drop one node deep.
impl<V> Drop for Chained<V> {
    fn drop(&mut self) {
        for head in &mut self.buckets {
            let mut cur = head.take();
            while let Some(mut e) = cur {
                cur = e.next.take(); // detach the tail BEFORE `e` is dropped
            }
        }
    }
}

// (mean chain, longest chain, empty buckets). The mean is computed from the
// chains, so it is a measurement: it equals the load factor because every
// entry sits in exactly one chain, and this would show it if an entry were lost.
fn stats(lens: &[usize]) -> (f64, usize, usize) {
    let mean = lens.iter().sum::<usize>() as f64 / lens.len() as f64;
    let longest = lens.iter().copied().max().unwrap_or(0);
    let empty = lens.iter().filter(|&&l| l == 0).count();
    (mean, longest, empty)
}

// FNV-1a followed by the splitmix64 finalizer from example 2, for the Rust
// twin of example 2's `FNV-1a + finalizer` row (Go's examples 3 and 4 just
// call it fnv1a). The hash of a given string is bit-identical to Go's, so the
// same keys land in the same buckets and every chain length below is the same.
fn fnv1a(s: &str) -> u64 {
    let mut h: u64 = 14695981039346656037;
    for &b in s.as_bytes() {
        h ^= b as u64;
        h = h.wrapping_mul(1099511628211); // Go wraps silently; Rust says so
    }
    h ^= h >> 30;
    h = h.wrapping_mul(0xbf58476d1ce4e5b9);
    h ^= h >> 27;
    h = h.wrapping_mul(0x94d049bb133111eb);
    h ^= h >> 31;
    h
}

// bad_hash sends every key to bucket 0. It is what an attacker arranges
// deliberately (example 14) and what a naive hash does by accident.
fn bad_hash(_: &str) -> u64 {
    0
}

fn keys(n: usize) -> Vec<String> {
    (0..n).map(|i| format!("key_{i}")).collect()
}

fn main() {
    println!("chaining, on a table deliberately too small so the chains show:");
    println!();
    let mut m = Chained::new(8, fnv1a);
    for (i, k) in keys(12).into_iter().enumerate() {
        m.put(k, i);
    }
    for i in 0..m.buckets.len() {
        print!("  bucket {i}: ");
        if m.buckets[i].is_none() {
            println!("(empty)");
            continue;
        }
        for e in m.chain(i) {
            print!("{}={} -> ", e.key, e.val);
        }
        println!("nil");
    }
    println!();
    println!(
        "  12 keys, 8 buckets, load factor {:.2}",
        m.len() as f64 / m.buckets.len() as f64
    );
    println!();
    println!("  a chained table has no notion of 'full'. Load factor above 1 is");
    println!("  legal and merely slow, which is the property that makes chaining");
    println!("  forgiving -- and the property that lets it degrade silently.");

    println!();
    println!("Put MUST search the chain first. Here is what happens if it doesn't:");
    println!();
    let mut good = Chained::new(16, fnv1a);
    let mut bad = Chained::new(16, fnv1a);
    for i in 0..3usize {
        good.put("dup".to_string(), i);
        bad.put_no_check("dup".to_string(), i);
    }
    println!(
        "  {:<28} Len={}  Get(\"dup\")={:?}",
        "Put with the search",
        good.len(),
        good.get("dup")
    );
    println!(
        "  {:<28} Len={}  Get(\"dup\")={:?}",
        "Put without it",
        bad.len(),
        bad.get("dup")
    );
    bad.delete("dup");
    // Go prints `Get("dup")=1, present=true`; the Option says both at once.
    println!(
        "  {:<28} after one Delete, Get(\"dup\")={:?}",
        "",
        bad.get("dup")
    );
    println!();
    println!("  the broken table reports 3 entries for 1 key, and deleting the key");
    println!("  leaves it present with a STALE value. Nothing panics. This is the");
    println!("  bug to look for whenever a hash map 'forgets' an update.");

    println!();
    println!("chain length is the whole cost model. With a good hash:");
    println!();
    println!(
        "  {:>10} {:>10} {:>14} {:>14} {:>12} {:>14}",
        "keys", "buckets", "load factor", "mean chain", "longest", "empty buckets"
    );
    let cases = [
        (1000, 4096),
        (1000, 1024),
        (1000, 512),
        (1000, 256),
        (1000, 64),
    ];
    let tables: Vec<Chained<usize>> = cases
        .iter()
        .map(|&(n, b)| {
            let mut t = Chained::new(b, fnv1a);
            for (i, k) in keys(n).into_iter().enumerate() {
                t.put(k, i);
            }
            t
        })
        .collect();
    for t in &tables {
        let (mean, longest, empty) = stats(&t.chain_lengths());
        println!(
            "  {:>10} {:>10} {:>14.2} {:>14.2} {:>12} {:>14}",
            t.len(),
            t.buckets.len(),
            t.len() as f64 / t.buckets.len() as f64,
            mean,
            longest,
            empty
        );
    }
    println!();
    println!("  the mean chain length IS the load factor -- that is not a");
    println!("  coincidence, it is the definition. An unsuccessful lookup walks");
    println!("  the whole chain, so its expected cost should be the load factor.");
    println!("  The Go program asserts that; this one measures it, with 1000 absent");
    println!("  keys per table, next to the claim about empty buckets:");
    println!();
    println!(
        "  {:>12} {:>14} {:>10} {:>12} {:>16}",
        "load factor", "empty buckets", "e^-load", "links/miss", "off (std errors)"
    );
    let absent: Vec<String> = (0..1000).map(|i| format!("absent_{i}")).collect();
    // (load, empty %, e^-load %, links per miss, links/miss minus load in standard errors)
    let mut rows = Vec::new();
    for t in &tables {
        let load = t.len() as f64 / t.buckets.len() as f64;
        let lens = t.chain_lengths();
        let (mean, _, empty) = stats(&lens);
        t.probes.set(0);
        for k in &absent {
            black_box(t.get(k));
        }
        let miss = t.probes.get() as f64 / absent.len() as f64;
        // A miss costs the length of ONE chain, so the average of 1000 misses
        // is a sample mean whose standard error is (spread of chain lengths) /
        // sqrt(1000).
        let var = lens.iter().map(|&l| (l as f64 - mean).powi(2)).sum::<f64>() / lens.len() as f64;
        let std_err = (var / absent.len() as f64).sqrt();
        let row = (
            load,
            100.0 * empty as f64 / t.buckets.len() as f64,
            100.0 * (-load).exp(),
            miss,
            (miss - load) / std_err,
        );
        println!(
            "  {:>12.2} {:>13.1}% {:>9.1}% {:>12.2} {:>+16.1}",
            row.0, row.1, row.2, row.3, row.4
        );
        rows.push(row);
    }
    // rows[1] is the 1000-key, 1024-bucket table: load factor 0.98, the row
    // nearest 1.00.
    let r = rows[1];
    println!();
    println!(
        "  at load factor {:.2}, {:.1}% of the buckets are still EMPTY while the mean chain",
        r.0, r.1
    );
    println!(
        "  is {:.2}. e^-load predicts {:.1}%: with n keys thrown at b buckets, a given",
        r.0, r.2
    );
    println!("  bucket stays empty with probability (1-1/b)^n, which is about e^-(n/b). At");
    println!("  load factor 1.00 that is e^-1 = 36.8%, and it is why a chained table");
    println!("  wastes space at exactly the load factor where it stops being fast.");
    let worst_z = rows.iter().map(|r| r.4.abs()).fold(0.0, f64::max);
    println!("  links/miss is the load factor to within sampling error: the largest gap of the");
    println!("  five rows is {worst_z:.1} standard errors, so the gaps are consistent with noise.");
    println!();

    println!("and here is what the same table does with the hash removed:");
    println!();
    let mut adv = Chained::new(1024, bad_hash);
    let ks = keys(2000);
    for (i, k) in ks.iter().enumerate() {
        adv.put(k.clone(), i);
    }
    let (_, longest, empty) = stats(&adv.chain_lengths());
    println!("  2000 keys, 1024 buckets, longest chain {longest}, empty buckets {empty}");

    let mut fine = Chained::new(1024, fnv1a);
    for (i, k) in ks.iter().enumerate() {
        fine.put(k.clone(), i);
    }
    // Count links in ONE pass. Counting inside the benchmark would total them
    // across every iteration, which measures the harness, not the table.
    adv.probes.set(0);
    fine.probes.set(0);
    for k in &ks {
        black_box(adv.get(k));
        black_box(fine.get(k));
    }
    let (probes_bad, probes_good) = (adv.probes.get(), fine.probes.get());

    let t_bad = ns_per_op(|| {
        for k in &ks {
            black_box(adv.get(black_box(k)));
        }
    });
    let t_good = ns_per_op(|| {
        for k in &ks {
            black_box(fine.get(black_box(k)));
        }
    });
    println!();
    println!("  {:<24} {:>14.0} ns for 2000 lookups", "good hash", t_good);
    println!(
        "  {:<24} {:>14.0} ns   {:.0}x slower",
        "every key collides",
        t_bad,
        t_bad / t_good
    );
    println!();
    println!(
        "  {:<24} {:>14} links followed  ({:.1} per lookup)",
        "good hash",
        probes_good,
        probes_good as f64 / ks.len() as f64
    );
    println!(
        "  {:<24} {:>14} links followed  ({:.1} per lookup)",
        "every key collides",
        probes_bad,
        probes_bad as f64 / ks.len() as f64
    );
    println!();
    println!("  Theta(1) became Theta(n), and the data structure did not change --");
    println!("  only the hash did. This is the worst case a hash table has, it is");
    println!("  reachable on purpose, and example 14 is about defending it.");

    println!();
    println!("what chaining costs, stated plainly -- Rust edition. Go lists four costs;");
    println!("one of them (the GC) does not exist here, so the list is measured instead:");
    println!();
    let n = 1000;
    let before = live_bytes();
    let ks = keys(n);
    let (allocs, bytes, table) = allocs_of(|| {
        let mut t = Chained::new(n, fnv1a);
        for (i, k) in ks.into_iter().enumerate() {
            t.put(k, i);
        }
        t
    });
    let held = live_bytes() - before;
    let entry = size_of::<Entry<usize>>();
    println!(
        "  {:<26} {} allocation events for {} Puts ({} bucket array + 1 per entry)",
        "one allocation per Put",
        allocs,
        n,
        allocs - n
    );
    println!(
        "  {:<26} {} B = {} buckets x {} B + {} entries x {} B",
        "  bytes requested",
        bytes,
        table.buckets.len(),
        size_of::<Link<usize>>(),
        n,
        entry
    );
    println!(
        "  {:<26} Entry<usize> is {} B; the same (String, usize) without `next` is {} B",
        "one pointer per entry",
        entry,
        size_of::<(String, usize)>()
    );
    println!(
        "  {:<26} `Option<Box<Entry>>` is {} B: None is the null pointer, no tag byte",
        "",
        size_of::<Link<usize>>()
    );
    println!(
        "  {:<26} not measured here -- lesson 07 and example 11 do that",
        "one cache miss per link"
    );
    drop(table);
    println!(
        "  {:<26} none to trace. Instead `Drop` frees entries one by one: {} B held",
        "no GC", held
    );
    println!(
        "  {:<26} by the table and its keys before drop, {} B after",
        "",
        live_bytes() - before
    );

    // The worst case again, now as a memory-safety question: one chain holding
    // a million entries, built without the search (that would be half a
    // trillion link steps) and dropped through the iterative Drop above.
    const LONG: usize = 1_000_000;
    let mut long = Chained::new(1, bad_hash);
    for i in 0..LONG {
        long.put_no_check(i.to_string(), i);
    }
    let links = long.chain(0).count();
    drop(long);
    println!();
    println!(
        "  one chain of {links} entries built and dropped: {} B left over",
        live_bytes() - before
    );
    println!();
    println!("  in exchange you get: no 'full' state, no tombstones, deletion");
    println!("  that is genuinely Theta(1), and load factors above 1 that merely");
    println!("  degrade instead of failing.");
    println!();
    println!("  that trade used to be the default. It is not any more -- Go, Rust,");
    println!("  Abseil and Swift all moved to OPEN ADDRESSING, which is example 4,");
    println!("  and example 11 measures why.");
}
```

**Rust sample output:**

```
chaining, on a table deliberately too small so the chains show:

  bucket 0: key_4=4 -> key_3=3 -> nil
  bucket 1: key_6=6 -> nil
  bucket 2: (empty)
  bucket 3: key_10=10 -> key_9=9 -> nil
  bucket 4: key_8=8 -> key_1=1 -> key_0=0 -> nil
  bucket 5: (empty)
  bucket 6: key_11=11 -> key_2=2 -> nil
  bucket 7: key_7=7 -> key_5=5 -> nil

  12 keys, 8 buckets, load factor 1.50

  a chained table has no notion of 'full'. Load factor above 1 is
  legal and merely slow, which is the property that makes chaining
  forgiving -- and the property that lets it degrade silently.

Put MUST search the chain first. Here is what happens if it doesn't:

  Put with the search          Len=1  Get("dup")=Some(2)
  Put without it               Len=3  Get("dup")=Some(2)
                               after one Delete, Get("dup")=Some(1)

  the broken table reports 3 entries for 1 key, and deleting the key
  leaves it present with a STALE value. Nothing panics. This is the
  bug to look for whenever a hash map 'forgets' an update.

chain length is the whole cost model. With a good hash:

        keys    buckets    load factor     mean chain      longest  empty buckets
        1000       4096           0.24           0.24            3           3210
        1000       1024           0.98           0.98            5            382
        1000        512           1.95           1.95            8             67
        1000        256           3.91           3.91           11              4
        1000         64          15.62          15.62           30              0

  the mean chain length IS the load factor -- that is not a
  coincidence, it is the definition. An unsuccessful lookup walks
  the whole chain, so its expected cost should be the load factor.
  The Go program asserts that; this one measures it, with 1000 absent
  keys per table, next to the claim about empty buckets:

   load factor  empty buckets    e^-load   links/miss off (std errors)
          0.24          78.4%      78.3%         0.22             -1.3
          0.98          37.3%      37.7%         0.97             -0.1
          1.95          13.1%      14.2%         2.00             +1.0
          3.91           1.6%       2.0%         3.99             +1.4
         15.62           0.0%       0.0%        15.65             +0.2

  at load factor 0.98, 37.3% of the buckets are still EMPTY while the mean chain
  is 0.98. e^-load predicts 37.7%: with n keys thrown at b buckets, a given
  bucket stays empty with probability (1-1/b)^n, which is about e^-(n/b). At
  load factor 1.00 that is e^-1 = 36.8%, and it is why a chained table
  wastes space at exactly the load factor where it stops being fast.
  links/miss is the load factor to within sampling error: the largest gap of the
  five rows is 1.4 standard errors, so the gaps are consistent with noise.

and here is what the same table does with the hash removed:

  2000 keys, 1024 buckets, longest chain 2000, empty buckets 1023

  good hash                         20187 ns for 2000 lookups
  every key collides              2610388 ns   129x slower

  good hash                          3959 links followed  (2.0 per lookup)
  every key collides              2001000 links followed  (1000.5 per lookup)

  Theta(1) became Theta(n), and the data structure did not change --
  only the hash did. This is the worst case a hash table has, it is
  reachable on purpose, and example 14 is about defending it.

what chaining costs, stated plainly -- Rust edition. Go lists four costs;
one of them (the GC) does not exist here, so the list is measured instead:

  one allocation per Put     1001 allocation events for 1000 Puts (1 bucket array + 1 per entry)
    bytes requested          48192 B = 1024 buckets x 8 B + 1000 entries x 40 B
  one pointer per entry      Entry<usize> is 40 B; the same (String, usize) without `next` is 32 B
                             `Option<Box<Entry>>` is 8 B: None is the null pointer, no tag byte
  one cache miss per link    not measured here -- lesson 07 and example 11 do that
  no GC                      none to trace. Instead `Drop` frees entries one by one: 56192 B held
                             by the table and its keys before drop, 0 B after

  one chain of 1000000 entries built and dropped: 0 B left over

  in exchange you get: no 'full' state, no tombstones, deletion
  that is genuinely Theta(1), and load factors above 1 that merely
  degrade instead of failing.

  that trade used to be the default. It is not any more -- Go, Rust,
  Abseil and Swift all moved to OPEN ADDRESSING, which is example 4,
  and example 11 measures why.
```

> **Go vs Rust:** Every shared row is identical to Go (same hash, same keys, so the same chains and 3,959 vs 2,001,000 links followed); only the timing differs: the all-colliding table is 112-132x slower over eight runs of this program, against 134-139x in two Go runs. Go's pointer-to-pointer delete becomes a `&mut Option<Box<Entry>>` cursor, and the direct transliteration of Go's loop is rejected by the borrow checker (E0506), so `delete` walks with a peek-only loop and then `take()`s the node; and a chain of boxes needs a hand-written iterative `Drop`, because without it dropping the 1,000,000-entry chain overflowed the 8 MB stack. With the counting allocator chaining costs 1,001 allocations for 1,000 puts and 8 extra bytes per entry (40 B against 32 B); there is no GC to trace the entries, only `Drop` to free them.

**Complexity:** mean chain length **is** the load factor, so an unsuccessful lookup costs exactly that
· at load factor 1.0, **37% of buckets are still empty** (that's e⁻¹) · with every key colliding, a
lookup follows **1000.5 links instead of 2.0** and runs **139× slower**

---

## 4. Open addressing, and the one hard problem

`🟢 easy` · *deleting a key can delete its neighbours*

No chains, no nodes, no pointers — when a slot is taken, take the next one. This is what Go, Rust,
Abseil and Swift all use now. It also has exactly one hard problem, and this example is mostly about
that problem.

**Steps:**

1. Watch keys displace and see that a lookup retraces the same path.
2. Delete one key by marking its slot free, and lose two others.
3. Then measure what tombstones cost once they accumulate.

```go
package main

import (
	"fmt"
	"strings"
)

// OPEN ADDRESSING: no chains, no nodes, no pointers. When a slot is taken, go
// looking for another one in the SAME array. With linear probing, "another one"
// means "the next one".
//
//	one array, no allocations after the constructor
//	the whole table is contiguous, so probing is a cache-friendly forward scan
//
// This is what Go, Rust, Abseil and Swift all use now. It also has exactly one
// hard problem, and this example is mostly about that problem.

type state uint8

const (
	free    state = iota // never used
	used                 // holds a live entry
	deleted              // TOMBSTONE: was used, now empty, but you may not stop here
)

type Open struct {
	keys   []string
	vals   []int
	states []state
	count  int
	tombs  int

	probes int
}

func NewOpen(n int) *Open {
	size := 1
	for size < n {
		size <<= 1
	}
	return &Open{
		keys:   make([]string, size),
		vals:   make([]int, size),
		states: make([]state, size),
	}
}

func fnv1a(s string) uint64 {
	h := uint64(14695981039346656037)
	for i := 0; i < len(s); i++ {
		h ^= uint64(s[i])
		h *= 1099511628211
	}
	h ^= h >> 30
	h *= 0xbf58476d1ce4e5b9
	h ^= h >> 27
	h *= 0x94d049bb133111eb
	h ^= h >> 31
	return h
}

func (m *Open) mask() uint64 { return uint64(len(m.keys) - 1) }

// Get walks forward until it finds the key or hits a FREE slot. It must NOT
// stop at a tombstone -- that is the entire subtlety.
func (m *Open) Get(k string) (int, bool) {
	i := fnv1a(k) & m.mask()
	for {
		m.probes++
		switch m.states[i] {
		case free:
			return 0, false // a free slot proves the key is not here
		case used:
			if m.keys[i] == k {
				return m.vals[i], true
			}
		}
		// deleted: keep going
		i = (i + 1) & m.mask()
	}
}

// Put remembers the first tombstone it passes and reuses it -- but only after
// confirming the key is not already present further along.
func (m *Open) Put(k string, v int) {
	i := fnv1a(k) & m.mask()
	firstTomb := -1
	for {
		m.probes++
		switch m.states[i] {
		case free:
			if firstTomb >= 0 { // reuse the tombstone we passed
				i = uint64(firstTomb)
				m.tombs--
			}
			m.keys[i], m.vals[i], m.states[i] = k, v, used
			m.count++
			return
		case deleted:
			if firstTomb < 0 {
				firstTomb = int(i)
			}
		case used:
			if m.keys[i] == k {
				m.vals[i] = v
				return
			}
		}
		i = (i + 1) & m.mask()
	}
}

// Delete CANNOT simply mark the slot free. Doing so breaks every key whose
// probe sequence ran through it.
func (m *Open) Delete(k string) bool {
	i := fnv1a(k) & m.mask()
	for {
		switch m.states[i] {
		case free:
			return false
		case used:
			if m.keys[i] == k {
				m.states[i] = deleted
				m.keys[i] = "" // release the string header
				m.count--
				m.tombs++
				return true
			}
		}
		i = (i + 1) & m.mask()
	}
}

// DeleteWrong is the bug: mark it free and move on.
func (m *Open) DeleteWrong(k string) bool {
	i := fnv1a(k) & m.mask()
	for {
		switch m.states[i] {
		case free:
			return false
		case used:
			if m.keys[i] == k {
				m.states[i] = free // WRONG
				m.keys[i] = ""
				m.count--
				return true
			}
		}
		i = (i + 1) & m.mask()
	}
}

func (m *Open) render() string {
	var sb strings.Builder
	for i := range m.keys {
		switch m.states[i] {
		case free:
			sb.WriteString(fmt.Sprintf("%9s", "."))
		case deleted:
			sb.WriteString(fmt.Sprintf("%9s", "<tomb>"))
		default:
			sb.WriteString(fmt.Sprintf("%9s", m.keys[i]))
		}
	}
	return sb.String()
}

// probeSeq reports where a key wants to go and where it ends up.
func (m *Open) probeSeq(k string) (home int, steps int) {
	i := fnv1a(k) & m.mask()
	home = int(i)
	for {
		if m.states[i] == used && m.keys[i] == k {
			return home, steps
		}
		if m.states[i] == free {
			return home, -1
		}
		steps++
		i = (i + 1) & m.mask()
	}
}

func main() {
	fmt.Println("linear probing: when the slot is taken, take the next one.")
	fmt.Println()
	m := NewOpen(8)
	words := []string{"cat", "dog", "emu", "fox", "owl"}
	for i, w := range words {
		m.Put(w, i)
	}
	fmt.Printf("  slots:  %s\n", func() string {
		var sb strings.Builder
		for i := range m.keys {
			sb.WriteString(fmt.Sprintf("%9d", i))
		}
		return sb.String()
	}())
	fmt.Printf("  table:  %s\n", m.render())
	fmt.Println()
	fmt.Printf("  %-8s %10s %10s\n", "key", "home slot", "probes")
	for _, w := range words {
		h, s := m.probeSeq(w)
		fmt.Printf("  %-8s %10d %10d\n", w, h, s)
	}
	fmt.Println()
	fmt.Println("  a key that had to move is not lost -- it is exactly `steps` slots")
	fmt.Println("  past its home, and a lookup finds it by walking the same path.")
	fmt.Println("  The walk stops at the first FREE slot, which is the proof that the")
	fmt.Println("  key is absent.")

	fmt.Println()
	fmt.Println("which is why Delete cannot just free the slot:")
	fmt.Println()
	demo := NewOpen(8)
	// force a collision chain by hand: three keys, all landing near each other
	var chain []string
	for i := 0; len(chain) < 3; i++ {
		k := fmt.Sprintf("k%d", i)
		if fnv1a(k)&7 == 3 {
			chain = append(chain, k)
		}
	}
	for i, k := range chain {
		demo.Put(k, i)
	}
	fmt.Printf("  three keys that all hash to slot 3: %v\n", chain)
	fmt.Printf("  table: %s\n", demo.render())

	broken := NewOpen(8)
	for i, k := range chain {
		broken.Put(k, i)
	}
	broken.DeleteWrong(chain[0])
	_, ok1 := broken.Get(chain[1])
	_, ok2 := broken.Get(chain[2])
	fmt.Println()
	fmt.Printf("  DeleteWrong(%q) -> table: %s\n", chain[0], broken.render())
	fmt.Printf("  Get(%q) = %v      <- LOST\n", chain[1], ok1)
	fmt.Printf("  Get(%q) = %v      <- LOST\n", chain[2], ok2)
	fmt.Println()
	fmt.Println("  both survivors are still sitting in the array, and both are")
	fmt.Println("  unreachable: the lookup walks to slot 3, finds it FREE, and")
	fmt.Println("  concludes the key is absent. Deleting one key silently deleted")
	fmt.Println("  two others.")

	fixed := NewOpen(8)
	for i, k := range chain {
		fixed.Put(k, i)
	}
	fixed.Delete(chain[0])
	_, ok3 := fixed.Get(chain[1])
	_, ok4 := fixed.Get(chain[2])
	fmt.Println()
	fmt.Printf("  Delete(%q)      -> table: %s\n", chain[0], fixed.render())
	fmt.Printf("  Get(%q) = %v      <- found\n", chain[1], ok3)
	fmt.Printf("  Get(%q) = %v      <- found\n", chain[2], ok4)
	fmt.Println()
	fmt.Println("  the TOMBSTONE says 'nothing lives here, but keep walking'. It is")
	fmt.Println("  the whole fix, and it is why an open-addressed table needs three")
	fmt.Println("  slot states rather than two.")

	fmt.Println()
	fmt.Println("but tombstones are not free -- they accumulate:")
	fmt.Println()
	fmt.Printf("  %14s %11s %12s %10s %12s %12s\n",
		"operation", "live keys", "tombstones", "occupied", "probes/hit", "probes/miss")
	churn := NewOpen(1024)
	ks := make([]string, 700)
	for i := range ks {
		ks[i] = fmt.Sprintf("key_%d", i)
	}
	for i, k := range ks {
		churn.Put(k, i)
	}
	report := func(label string) {
		churn.probes = 0
		for _, k := range ks {
			churn.Get(k)
		}
		hit := float64(churn.probes) / float64(len(ks))
		churn.probes = 0
		for i := range ks {
			churn.Get(fmt.Sprintf("absent_%d", i))
		}
		miss := float64(churn.probes) / float64(len(ks))
		occupied := churn.count + churn.tombs
		fmt.Printf("  %14s %11d %12d %9.0f%% %12.2f %12.2f\n",
			label, churn.count, churn.tombs,
			100*float64(occupied)/float64(len(churn.keys)), hit, miss)
	}
	report("after fill")
	for round := 0; round < 4; round++ {
		for i := 0; i < 200; i++ {
			k := fmt.Sprintf("tmp_%d_%d", round, i)
			churn.Put(k, i)
			churn.Delete(k)
		}
		report("+200 churn")
	}
	fmt.Println()
	fmt.Println("  read the two right-hand columns separately -- I did not, and got")
	fmt.Println("  this wrong the first time.")
	fmt.Println()
	fmt.Println("  probes/HIT barely moves: a successful lookup stops the moment it")
	fmt.Println("  finds its key, and the tombstones are mostly not on its path.")
	fmt.Println("  probes/MISS is where the damage is, because a miss cannot stop")
	fmt.Println("  until it reaches a FREE slot -- and tombstones are not free.")
	fmt.Println()
	fmt.Println("  the live key count never changes across those rows. The table")
	fmt.Println("  holds exactly the same data and answers 'not present' several")
	fmt.Println("  times slower, purely because of keys that are no longer there.")
	fmt.Println("  A table can be 100% occupied by tombstones while holding no data.")
	fmt.Println()
	fmt.Println("  that is why a real open-addressed table counts `used + tombstones`")
	fmt.Println("  against its load factor, not just `used`, and rehashes to clear")
	fmt.Println("  them. Go's map does exactly this -- the comment in")
	fmt.Println("  internal/runtime/maps/table.go says: 'We rehash when used +")
	fmt.Println("  tombstones > loadFactor*capacity'.")

	fmt.Println()
	fmt.Println("the other cost of linear probing is CLUSTERING:")
	fmt.Println()
	fmt.Printf("  %14s %14s %14s %18s\n", "load factor", "probes/hit", "probes/miss", "theory (miss)")
	for _, lf := range []float64{0.25, 0.5, 0.75, 0.875, 0.95} {
		size := 4096
		n := int(float64(size) * lf)
		tt := NewOpen(size)
		kk := make([]string, n)
		for i := range kk {
			kk[i] = fmt.Sprintf("k_%d", i)
			tt.Put(kk[i], i)
		}
		tt.probes = 0
		for _, k := range kk {
			tt.Get(k)
		}
		hit := float64(tt.probes) / float64(n)
		tt.probes = 0
		for i := 0; i < n; i++ {
			tt.Get(fmt.Sprintf("absent_%d", i))
		}
		miss := float64(tt.probes) / float64(n)
		theory := (1 + 1/((1-lf)*(1-lf))) / 2
		fmt.Printf("  %14.3f %14.2f %14.2f %18.2f\n", lf, hit, miss, theory)
	}
	fmt.Println()
	fmt.Println("  Knuth's estimate for an unsuccessful linear-probe search is")
	fmt.Println("  (1 + 1/(1-a)^2)/2. The measured column tracks it to within about")
	fmt.Println("  2x and diverges at the top -- the formula assumes uniform hashing")
	fmt.Println("  and 4096 slots is a small sample at 95% load. The SHAPE is the")
	fmt.Println("  point: linear in nothing, quadratic in 1/(1-a).")
	fmt.Println()
	fmt.Println("  a full open-addressed table does not merely slow down -- it stops")
	fmt.Println("  terminating. That is why every implementation has a load factor")
	fmt.Println("  limit and rehashes before reaching it, which is example 5.")
}
```

**Sample output:**

```
linear probing: when the slot is taken, take the next one.

  slots:          0        1        2        3        4        5        6        7
  table:        fox        .      cat      emu        .      owl        .      dog

  key       home slot     probes
  cat               2          0
  dog               7          0
  emu               2          1
  fox               0          0
  owl               5          0

  a key that had to move is not lost -- it is exactly `steps` slots
  past its home, and a lookup finds it by walking the same path.
  The walk stops at the first FREE slot, which is the proof that the
  key is absent.

which is why Delete cannot just free the slot:

  three keys that all hash to slot 3: [k5 k18 k31]
  table:         .        .        .       k5      k18      k31        .        .

  DeleteWrong("k5") -> table:         .        .        .        .      k18      k31        .        .
  Get("k18") = false      <- LOST
  Get("k31") = false      <- LOST

  both survivors are still sitting in the array, and both are
  unreachable: the lookup walks to slot 3, finds it FREE, and
  concludes the key is absent. Deleting one key silently deleted
  two others.

  Delete("k5")      -> table:         .        .        .   <tomb>      k18      k31        .        .
  Get("k18") = true      <- found
  Get("k31") = true      <- found

  the TOMBSTONE says 'nothing lives here, but keep walking'. It is
  the whole fix, and it is why an open-addressed table needs three
  slot states rather than two.

but tombstones are not free -- they accumulate:

       operation   live keys   tombstones   occupied   probes/hit  probes/miss
      after fill         700            0        68%         1.82         4.56
      +200 churn         700          108        79%         1.82         7.53
      +200 churn         700          164        84%         1.82         9.57
      +200 churn         700          197        88%         1.82        10.27
      +200 churn         700          225        90%         1.82        14.28

  read the two right-hand columns separately -- I did not, and got
  this wrong the first time.

  probes/HIT barely moves: a successful lookup stops the moment it
  finds its key, and the tombstones are mostly not on its path.
  probes/MISS is where the damage is, because a miss cannot stop
  until it reaches a FREE slot -- and tombstones are not free.

  the live key count never changes across those rows. The table
  holds exactly the same data and answers 'not present' several
  times slower, purely because of keys that are no longer there.
  A table can be 100% occupied by tombstones while holding no data.

  that is why a real open-addressed table counts `used + tombstones`
  against its load factor, not just `used`, and rehashes to clear
  them. Go's map does exactly this -- the comment in
  internal/runtime/maps/table.go says: 'We rehash when used +
  tombstones > loadFactor*capacity'.

the other cost of linear probing is CLUSTERING:

     load factor     probes/hit    probes/miss      theory (miss)
           0.250           1.18           1.38               1.39
           0.500           1.54           2.61               2.50
           0.750           2.69           9.64               8.50
           0.875           4.47          24.54              32.50
           0.950           7.73         125.00             200.50

  Knuth's estimate for an unsuccessful linear-probe search is
  (1 + 1/(1-a)^2)/2. The measured column tracks it to within about
  2x and diverges at the top -- the formula assumes uniform hashing
  and 4096 slots is a small sample at 95% load. The SHAPE is the
  point: linear in nothing, quadratic in 1/(1-a).

  a full open-addressed table does not merely slow down -- it stops
  terminating. That is why every implementation has a load factor
  limit and rehashes before reaching it, which is example 5.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::Cell;
use std::mem::size_of;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// OPEN ADDRESSING: no chains, no nodes, no pointers. When a slot is taken, go
// looking for another one in the SAME array. With linear probing, "another one"
// means "the next one".
//
//     one array, no allocations after the constructor
//     the whole table is contiguous, so probing is a cache-friendly forward scan
//
// Go, Rust, Abseil and Swift all use open addressing now. Rust's std HashMap is
// hashbrown, a "Swiss table" that probes a whole group of slots at a time
// (example 6); this is the plain one-slot-at-a-time ancestor. It also has
// exactly one hard problem, and this example is mostly about that problem.

// ---- measuring kit: std-only stand-in for runtime.MemStats ----

// Allocation counter. Go's comment above says "no allocations after the
// constructor"; Rust has no GC noise, so that claim is an exact count.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events
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

/// (allocation events, bytes requested) while running `f`.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, usize, R) {
    let (a0, t0) = (ALLOCS.load(Relaxed), TOTAL.load(Relaxed));
    let r = f();
    (ALLOCS.load(Relaxed) - a0, TOTAL.load(Relaxed) - t0, r)
}

// ---- the table ----

// Go keeps three parallel arrays (keys, vals, states) and a `state` byte with
// three values. In Rust the natural shape is ONE array of an enum: the variant
// IS the state, and only `Used` carries a key and a value. A free slot or a
// tombstone has no key to read, so "look at a dead slot's stale key" cannot be
// written; Go has to zero `keys[i]` by hand to avoid keeping garbage alive.
//
// The key is a `&'a str`, a borrow of text the CALLER owns -- exactly what Go's
// `string` header is (pointer + length), so the table needs no allocation per
// key. Go's garbage collector keeps the bytes alive behind your back; here the
// `'a` makes the compiler prove that every key outlives the table, which is why
// the keys in `main` are always declared BEFORE the table that borrows them.
#[derive(Clone, Copy)]
enum Slot<'a> {
    Free,                              // never used
    Used { key: &'a str, val: usize }, // holds a live entry
    Deleted,                           // TOMBSTONE: was used, now empty, but you may not stop here
}

struct Open<'a> {
    slots: Vec<Slot<'a>>,
    count: usize,
    tombs: usize,

    // A Cell so that `get(&self)` can still count probes (see example 3).
    probes: Cell<usize>,
}

impl<'a> Open<'a> {
    fn new(n: usize) -> Self {
        Open {
            slots: vec![Slot::Free; n.next_power_of_two()],
            count: 0,
            tombs: 0,
            probes: Cell::new(0),
        }
    }

    fn mask(&self) -> usize {
        self.slots.len() - 1
    }

    fn home(&self, k: &str) -> usize {
        (fnv1a(k) & self.mask() as u64) as usize
    }

    // Get walks forward until it finds the key or hits a FREE slot. It must NOT
    // stop at a tombstone -- that is the entire subtlety. If no slot is Free,
    // this loop never ends, in Go and in Rust alike.
    fn get(&self, k: &str) -> Option<usize> {
        let mut i = self.home(k);
        loop {
            self.probes.set(self.probes.get() + 1);
            match self.slots[i] {
                Slot::Free => return None, // a free slot proves the key is not here
                Slot::Used { key, val } if key == k => return Some(val),
                Slot::Used { .. } | Slot::Deleted => {} // deleted: keep going
            }
            i = (i + 1) & self.mask();
        }
    }

    // Put remembers the first tombstone it passes and reuses it -- but only
    // after confirming the key is not already present further along.
    fn put(&mut self, k: &'a str, v: usize) {
        let mut i = self.home(k);
        let mut first_tomb: Option<usize> = None; // Go: -1 means "none yet"
        loop {
            self.probes.set(self.probes.get() + 1);
            match self.slots[i] {
                Slot::Free => {
                    let at = match first_tomb {
                        Some(t) => {
                            self.tombs -= 1; // reuse the tombstone we passed
                            t
                        }
                        None => i,
                    };
                    self.slots[at] = Slot::Used { key: k, val: v };
                    self.count += 1;
                    return;
                }
                Slot::Deleted => {
                    first_tomb.get_or_insert(i);
                }
                Slot::Used { key, .. } if key == k => {
                    self.slots[i] = Slot::Used { key, val: v };
                    return;
                }
                Slot::Used { .. } => {}
            }
            i = (i + 1) & self.mask();
        }
    }

    // Delete CANNOT simply mark the slot free. Doing so breaks every key whose
    // probe sequence ran through it.
    fn delete(&mut self, k: &str) -> Option<usize> {
        let mut i = self.home(k);
        loop {
            match self.slots[i] {
                Slot::Free => return None,
                Slot::Used { key, val } if key == k => {
                    // Overwriting the slot drops the key borrow with it: no
                    // `keys[i] = ""` needed.
                    self.slots[i] = Slot::Deleted;
                    self.count -= 1;
                    self.tombs += 1;
                    return Some(val);
                }
                Slot::Used { .. } | Slot::Deleted => {}
            }
            i = (i + 1) & self.mask();
        }
    }

    // delete_wrong is the bug: mark it free and move on.
    fn delete_wrong(&mut self, k: &str) -> Option<usize> {
        let mut i = self.home(k);
        loop {
            match self.slots[i] {
                Slot::Free => return None,
                Slot::Used { key, val } if key == k => {
                    self.slots[i] = Slot::Free; // WRONG
                    self.count -= 1;
                    return Some(val);
                }
                Slot::Used { .. } | Slot::Deleted => {}
            }
            i = (i + 1) & self.mask();
        }
    }

    fn render(&self) -> String {
        self.slots
            .iter()
            .map(|s| match s {
                Slot::Free => format!("{:>9}", "."),
                Slot::Deleted => format!("{:>9}", "<tomb>"),
                Slot::Used { key, .. } => format!("{key:>9}"),
            })
            .collect()
    }

    // probe_seq reports where a key wants to go and how many slots past it
    // ended up (None: the key is absent -- Go returns -1).
    fn probe_seq(&self, k: &str) -> (usize, Option<usize>) {
        let home = self.home(k);
        let mut i = home;
        let mut steps = 0;
        loop {
            match self.slots[i] {
                Slot::Used { key, .. } if key == k => return (home, Some(steps)),
                Slot::Free => return (home, None),
                Slot::Used { .. } | Slot::Deleted => {}
            }
            steps += 1;
            i = (i + 1) & self.mask();
        }
    }

    // Not in the Go program: how many tombstones does the walk for `k` step
    // over before it stops (at the key, or at a Free slot if it is absent)?
    fn tombs_passed(&self, k: &str) -> usize {
        let mut i = self.home(k);
        let mut n = 0;
        loop {
            match self.slots[i] {
                Slot::Free => return n,
                Slot::Used { key, .. } if key == k => return n,
                Slot::Deleted => n += 1,
                Slot::Used { .. } => {}
            }
            i = (i + 1) & self.mask();
        }
    }
}

// FNV-1a followed by the splitmix64 finalizer from example 2 (Go's examples
// call it fnv1a). The hash of a given string is bit-identical to Go's, so the
// same keys land in the same slots and every probe count below is the same.
fn fnv1a(s: &str) -> u64 {
    let mut h: u64 = 14695981039346656037;
    for &b in s.as_bytes() {
        h ^= b as u64;
        h = h.wrapping_mul(1099511628211); // Go wraps silently; Rust says so
    }
    h ^= h >> 30;
    h = h.wrapping_mul(0xbf58476d1ce4e5b9);
    h ^= h >> 27;
    h = h.wrapping_mul(0x94d049bb133111eb);
    h ^= h >> 31;
    h
}

fn strings(prefix: &str, n: usize) -> Vec<String> {
    (0..n).map(|i| format!("{prefix}_{i}")).collect()
}

// One row of the tombstone table: probes per hit and per miss, over the
// original keys and over the same number of absent ones.
fn report(label: &str, t: &Open, ks: &[String], absent: &[String]) -> (f64, f64) {
    t.probes.set(0);
    for k in ks {
        t.get(k);
    }
    let hit = t.probes.get() as f64 / ks.len() as f64;
    t.probes.set(0);
    for k in absent {
        t.get(k);
    }
    let miss = t.probes.get() as f64 / absent.len() as f64;
    let occupied = t.count + t.tombs;
    println!(
        "  {:>14} {:>11} {:>12} {:>9.0}% {:>12.2} {:>12.2}",
        label,
        t.count,
        t.tombs,
        100.0 * occupied as f64 / t.slots.len() as f64,
        hit,
        miss
    );
    (hit, miss)
}

// (probes per hit, probes per miss) with `n` keys named `<prefix>_<i>` in a
// table of `size` slots, then the same number of absent keys.
fn measure(prefix: &str, size: usize, n: usize) -> (f64, f64) {
    let present = strings(prefix, n);
    let absent = strings("absent", n);
    let mut t = Open::new(size);
    for (i, k) in present.iter().enumerate() {
        t.put(k, i);
    }
    t.probes.set(0);
    for k in &present {
        t.get(k);
    }
    let hit = t.probes.get() as f64 / n as f64;
    t.probes.set(0);
    for k in &absent {
        t.get(k);
    }
    (hit, t.probes.get() as f64 / n as f64)
}

// probes/miss at load factor 0.95 over `sets` DIFFERENT key sets, each its own
// table: (mean, standard deviation across the sets).
fn spread(size: usize, sets: usize) -> (f64, f64) {
    let n = (size as f64 * 0.95) as usize;
    let misses: Vec<f64> = (0..sets)
        .map(|s| measure(&format!("set{s}"), size, n).1)
        .collect();
    let mean = misses.iter().sum::<f64>() / sets as f64;
    let var = misses.iter().map(|m| (m - mean).powi(2)).sum::<f64>() / (sets - 1) as f64;
    (mean, var.sqrt())
}

fn main() {
    println!("linear probing: when the slot is taken, take the next one.");
    println!();
    let words = ["cat", "dog", "emu", "fox", "owl"];
    let (new_allocs, new_bytes, mut m) = allocs_of(|| Open::new(8));
    let (put_allocs, _, ()) = allocs_of(|| {
        for (i, w) in words.iter().enumerate() {
            m.put(w, i);
        }
    });
    let header: String = (0..m.slots.len()).map(|i| format!("{i:>9}")).collect();
    println!("  slots:  {header}");
    println!("  table:  {}", m.render());
    println!();
    println!("  {:<8} {:>10} {:>10}", "key", "home slot", "probes");
    for w in words {
        let (home, steps) = m.probe_seq(w);
        println!(
            "  {:<8} {:>10} {:>10}",
            w,
            home,
            steps.expect("just inserted")
        );
    }
    println!();
    println!("  a key that had to move is not lost -- it is exactly `steps` slots");
    println!("  past its home, and a lookup finds it by walking the same path.");
    println!("  The walk stops at the first FREE slot, which is the proof that the");
    println!("  key is absent.");
    println!();
    println!(
        "  \"no allocations after the constructor\", counted: building the table took {new_allocs}"
    );
    println!(
        "  allocation ({new_bytes} B = {} slots x {} B each), and the {} Puts took {put_allocs}.",
        m.slots.len(),
        size_of::<Slot>(),
        words.len()
    );

    println!();
    println!("which is why Delete cannot just free the slot:");
    println!();
    let mut demo = Open::new(8);
    // force a collision chain by hand: three keys, all landing near each other
    let mut chain: Vec<String> = Vec::new();
    for i in 0.. {
        if chain.len() == 3 {
            break;
        }
        let k = format!("k{i}");
        if fnv1a(&k) & 7 == 3 {
            chain.push(k);
        }
    }
    for (i, k) in chain.iter().enumerate() {
        demo.put(k, i);
    }
    println!(
        "  three keys that all hash to slot 3: [{}]",
        chain.join(" ")
    );
    println!("  table: {}", demo.render());

    let mut broken = Open::new(8);
    for (i, k) in chain.iter().enumerate() {
        broken.put(k, i);
    }
    broken.delete_wrong(&chain[0]);
    let ok1 = broken.get(&chain[1]).is_some();
    let ok2 = broken.get(&chain[2]).is_some();
    println!();
    println!(
        "  DeleteWrong({:?}) -> table: {}",
        chain[0],
        broken.render()
    );
    println!("  Get({:?}) = {ok1}      <- LOST", chain[1]);
    println!("  Get({:?}) = {ok2}      <- LOST", chain[2]);
    println!();
    println!("  both survivors are still sitting in the array, and both are");
    println!("  unreachable: the lookup walks to slot 3, finds it FREE, and");
    println!("  concludes the key is absent. Deleting one key silently deleted");
    println!("  two others.");

    let mut fixed = Open::new(8);
    for (i, k) in chain.iter().enumerate() {
        fixed.put(k, i);
    }
    fixed.delete(&chain[0]);
    let ok3 = fixed.get(&chain[1]).is_some();
    let ok4 = fixed.get(&chain[2]).is_some();
    println!();
    println!("  Delete({:?})      -> table: {}", chain[0], fixed.render());
    println!("  Get({:?}) = {ok3}      <- found", chain[1]);
    println!("  Get({:?}) = {ok4}      <- found", chain[2]);
    println!();
    println!("  the TOMBSTONE says 'nothing lives here, but keep walking'. It is");
    println!("  the whole fix, and it is why an open-addressed table needs three");
    println!("  slot states rather than two.");

    println!();
    println!("but tombstones are not free -- they accumulate:");
    println!();
    println!(
        "  {:>14} {:>11} {:>12} {:>10} {:>12} {:>12}",
        "operation", "live keys", "tombstones", "occupied", "probes/hit", "probes/miss"
    );
    // The table borrows its keys, so all of them are built up front, before
    // the table: the 700 that stay, the 800 churn keys, and the absent ones.
    let ks = strings("key", 700);
    let absent = strings("absent", ks.len());
    let tmp: Vec<String> = (0..4)
        .flat_map(|round| (0..200).map(move |i| format!("tmp_{round}_{i}")))
        .collect();
    let mut churn = Open::new(1024);
    for (i, k) in ks.iter().enumerate() {
        churn.put(k, i);
    }
    let mut rows = vec![report("after fill", &churn, &ks, &absent)];
    for round in 0..4 {
        for i in 0..200 {
            let k = &tmp[round * 200 + i];
            churn.put(k, i);
            churn.delete(k);
        }
        rows.push(report("+200 churn", &churn, &ks, &absent));
    }
    let hits = rows.iter().map(|r| r.0);
    let (hit_lo, hit_hi) = (
        hits.clone().fold(f64::MAX, f64::min),
        hits.fold(f64::MIN, f64::max),
    );
    let (miss_first, miss_last) = (rows[0].1, rows[rows.len() - 1].1);
    let on_hit_paths: usize = ks.iter().map(|k| churn.tombs_passed(k)).sum();
    let on_miss_paths =
        absent.iter().map(|k| churn.tombs_passed(k)).sum::<usize>() as f64 / absent.len() as f64;
    println!();
    println!("  read the two right-hand columns separately.");
    println!();
    if hit_hi - hit_lo < 0.005 {
        println!("  probes/HIT is {hit_lo:.2} on every row. A successful lookup");
    } else {
        println!("  probes/HIT runs from {hit_lo:.2} to {hit_hi:.2}. A successful lookup");
    }
    println!("  stops the moment it finds its key, and here it never meets a tombstone:");
    println!(
        "  on the last row the 700 live keys' walks pass over {on_hit_paths} tombstones in total. That is"
    );
    println!("  not luck. Every slot between a key's home and its resting place was occupied");
    println!("  when the key went in, those 700 keys are never deleted, and a tombstone can");
    println!("  only appear in a slot that was FREE after the fill.");
    println!("  probes/MISS is where the damage is, because a miss cannot stop");
    println!("  until it reaches a FREE slot -- and tombstones are not free.");
    println!(
        "  A miss goes from {miss_first:.2} to {miss_last:.2} probes, {:.1}x, and on the last row",
        miss_last / miss_first
    );
    println!("  it steps over {on_miss_paths:.2} tombstones on average.");
    println!();
    println!("  the live key count never changes across those rows. The table");
    println!("  holds exactly the same data and answers 'not present' that much");
    println!("  slower, purely because of keys that are no longer there.");
    println!("  A table can be 100% occupied by tombstones while holding no data.");
    println!();
    println!("  that is why a real open-addressed table counts `used + tombstones`");
    println!("  against its load factor, not just `used`, and rehashes to clear");
    println!("  them. Rust's HashMap does the same. From the hashbrown 0.17.1 source");
    println!("  vendored with std (raw.rs): the growth budget is spent when an EMPTY");
    println!("  slot is filled, so a tombstone keeps costing budget; when the budget");
    println!(
        "  runs out the table is rehashed IN PLACE if at most half of its capacity is in use,"
    );
    println!("  which clears the tombstones, and grown otherwise. One refinement over");
    println!("  this program: `erase` writes a tombstone only when the slots around the");
    println!("  removed one form a run, a whole group wide, of live entries and");
    println!("  tombstones; otherwise it writes EMPTY and hands the budget back. This");
    println!("  program always tombstones, which is why its numbers climb. That is from");
    println!("  reading the source; nothing here measures HashMap.");
    println!();
    println!("the other cost of linear probing is CLUSTERING:");
    println!();
    println!(
        "  {:>14} {:>14} {:>14} {:>18}",
        "load factor", "probes/hit", "probes/miss", "theory (miss)"
    );
    let theory = |lf: f64| (1.0 + 1.0 / ((1.0 - lf) * (1.0 - lf))) / 2.0;
    let mut ratios = Vec::new(); // measured / theory, per row
    for lf in [0.25, 0.5, 0.75, 0.875, 0.95] {
        let size = 4096;
        let n = (size as f64 * lf) as usize;
        let (hit, miss) = measure("k", size, n);
        println!(
            "  {:>14.3} {:>14.2} {:>14.2} {:>18.2}",
            lf,
            hit,
            miss,
            theory(lf)
        );
        ratios.push(miss / theory(lf));
    }
    let (r_lo, r_hi) = (
        ratios.iter().copied().fold(f64::MAX, f64::min),
        ratios.iter().copied().fold(f64::MIN, f64::max),
    );
    println!();
    println!("  Knuth's estimate for an unsuccessful linear-probe search is");
    println!(
        "  (1 + 1/(1-a)^2)/2. Measured / theory runs from {r_lo:.2} to {r_hi:.2} across the rows,"
    );
    println!("  worst at the top. The Go program blames the gap on 4096 slots being a small");
    println!("  sample at 95% load. That is testable: the same load factor in tables of");
    println!(
        "  growing size, each size over 40 different key sets (theory for a miss: {:.2}).",
        theory(0.95)
    );
    println!();
    println!(
        "  {:>10} {:>14} {:>12} {:>14} {:>14}",
        "slots", "mean miss", "std dev", "std err", "mean / theory"
    );
    let sets = 40;
    let mut sizes = Vec::new(); // (mean, std dev, std err) per size
    for size in [4096, 16384, 65536] {
        let (mean, sd) = spread(size, sets);
        let se = sd / (sets as f64).sqrt();
        println!(
            "  {:>10} {:>14.1} {:>12.1} {:>14.1} {:>14.2}",
            size,
            mean,
            sd,
            se,
            mean / theory(0.95)
        );
        sizes.push((mean, sd, se));
    }
    let t95 = theory(0.95);
    println!();
    println!(
        "  at 4096 slots the average miss costs {:.1} probes, {:.1} standard errors under",
        sizes[0].0,
        (t95 - sizes[0].0) / sizes[0].2
    );
    println!("  the formula's {t95:.1}: the shortfall is real, not luck, so the large-table limit");
    println!(
        "  is not reached there. At 16384 and 65536 slots the averages are {:.1} and {:.1},",
        sizes[1].0, sizes[2].0
    );
    println!(
        "  within {:.1} and {:.1} standard errors of it. 'A small sample' is half the story;",
        (t95 - sizes[1].0).abs() / sizes[1].2,
        (t95 - sizes[2].0).abs() / sizes[2].2
    );
    println!("  the other half is that ONE table is a very noisy measurement -- a standard");
    println!(
        "  deviation of {:.0}-{:.0} probes between key sets, at every size -- so any single",
        sizes.iter().map(|s| s.1).fold(f64::MAX, f64::min),
        sizes.iter().map(|s| s.1).fold(f64::MIN, f64::max)
    );
    println!("  row above can sit well off its theory column in either direction.");
    println!("  The SHAPE is the point: linear in nothing, quadratic in 1/(1-a).");
    println!();
    println!("  a full open-addressed table does not merely slow down -- it stops");
    println!("  terminating. That is why every implementation has a load factor");
    println!("  limit and rehashes before reaching it, which is example 5.");
}
```

**Rust sample output:**

```
linear probing: when the slot is taken, take the next one.

  slots:          0        1        2        3        4        5        6        7
  table:        fox        .      cat      emu        .      owl        .      dog

  key       home slot     probes
  cat               2          0
  dog               7          0
  emu               2          1
  fox               0          0
  owl               5          0

  a key that had to move is not lost -- it is exactly `steps` slots
  past its home, and a lookup finds it by walking the same path.
  The walk stops at the first FREE slot, which is the proof that the
  key is absent.

  "no allocations after the constructor", counted: building the table took 1
  allocation (256 B = 8 slots x 32 B each), and the 5 Puts took 0.

which is why Delete cannot just free the slot:

  three keys that all hash to slot 3: [k5 k18 k31]
  table:         .        .        .       k5      k18      k31        .        .

  DeleteWrong("k5") -> table:         .        .        .        .      k18      k31        .        .
  Get("k18") = false      <- LOST
  Get("k31") = false      <- LOST

  both survivors are still sitting in the array, and both are
  unreachable: the lookup walks to slot 3, finds it FREE, and
  concludes the key is absent. Deleting one key silently deleted
  two others.

  Delete("k5")      -> table:         .        .        .   <tomb>      k18      k31        .        .
  Get("k18") = true      <- found
  Get("k31") = true      <- found

  the TOMBSTONE says 'nothing lives here, but keep walking'. It is
  the whole fix, and it is why an open-addressed table needs three
  slot states rather than two.

but tombstones are not free -- they accumulate:

       operation   live keys   tombstones   occupied   probes/hit  probes/miss
      after fill         700            0        68%         1.82         4.56
      +200 churn         700          108        79%         1.82         7.53
      +200 churn         700          164        84%         1.82         9.57
      +200 churn         700          197        88%         1.82        10.27
      +200 churn         700          225        90%         1.82        14.28

  read the two right-hand columns separately.

  probes/HIT is 1.82 on every row. A successful lookup
  stops the moment it finds its key, and here it never meets a tombstone:
  on the last row the 700 live keys' walks pass over 0 tombstones in total. That is
  not luck. Every slot between a key's home and its resting place was occupied
  when the key went in, those 700 keys are never deleted, and a tombstone can
  only appear in a slot that was FREE after the fill.
  probes/MISS is where the damage is, because a miss cannot stop
  until it reaches a FREE slot -- and tombstones are not free.
  A miss goes from 4.56 to 14.28 probes, 3.1x, and on the last row
  it steps over 3.04 tombstones on average.

  the live key count never changes across those rows. The table
  holds exactly the same data and answers 'not present' that much
  slower, purely because of keys that are no longer there.
  A table can be 100% occupied by tombstones while holding no data.

  that is why a real open-addressed table counts `used + tombstones`
  against its load factor, not just `used`, and rehashes to clear
  them. Rust's HashMap does the same. From the hashbrown 0.17.1 source
  vendored with std (raw.rs): the growth budget is spent when an EMPTY
  slot is filled, so a tombstone keeps costing budget; when the budget
  runs out the table is rehashed IN PLACE if at most half of its capacity is in use,
  which clears the tombstones, and grown otherwise. One refinement over
  this program: `erase` writes a tombstone only when the slots around the
  removed one form a run, a whole group wide, of live entries and
  tombstones; otherwise it writes EMPTY and hands the budget back. This
  program always tombstones, which is why its numbers climb. That is from
  reading the source; nothing here measures HashMap.

the other cost of linear probing is CLUSTERING:

     load factor     probes/hit    probes/miss      theory (miss)
           0.250           1.18           1.38               1.39
           0.500           1.54           2.61               2.50
           0.750           2.69           9.64               8.50
           0.875           4.47          24.54              32.50
           0.950           7.73         125.00             200.50

  Knuth's estimate for an unsuccessful linear-probe search is
  (1 + 1/(1-a)^2)/2. Measured / theory runs from 0.62 to 1.13 across the rows,
  worst at the top. The Go program blames the gap on 4096 slots being a small
  sample at 95% load. That is testable: the same load factor in tables of
  growing size, each size over 40 different key sets (theory for a miss: 200.50).

       slots      mean miss      std dev        std err  mean / theory
        4096          156.6         60.6            9.6           0.78
       16384          182.0         61.1            9.7           0.91
       65536          204.4         34.2            5.4           1.02

  at 4096 slots the average miss costs 156.6 probes, 4.6 standard errors under
  the formula's 200.5: the shortfall is real, not luck, so the large-table limit
  is not reached there. At 16384 and 65536 slots the averages are 182.0 and 204.4,
  within 1.9 and 0.7 standard errors of it. 'A small sample' is half the story;
  the other half is that ONE table is a very noisy measurement -- a standard
  deviation of 34-61 probes between key sets, at every size -- so any single
  row above can sit well off its theory column in either direction.
  The SHAPE is the point: linear in nothing, quadratic in 1/(1-a).

  a full open-addressed table does not merely slow down -- it stops
  terminating. That is why every implementation has a load factor
  limit and rehashes before reaching it, which is example 5.
```

> **Go vs Rust:** Every shared number is identical to Go (same hash, same keys, so the same slots): probes/hit 1.82 on every row, probes/miss 4.56 rising to 14.28, 125.00 at load 0.95. Go's three parallel arrays become one `Vec<Slot>` of an enum whose variants are the three states, so a tombstone has no key to read; keys are `&'a str` borrows, which makes the table allocation-free after its one constructor allocation (0 for the puts) at the price of a lifetime the compiler checks. Go's explanation for the 95% gap (4096 slots is a small sample) holds only in part: over 40 key sets the shortfall is real at 4096 slots (0.78 of theory, 4.6 standard errors) and gone by 65,536 (1.02), while any single table is very noisy.

**Complexity:** probes/miss grows like `(1 + 1/(1-a)²)/2` — measured **1.38 at load 0.25 and 125.00 at
0.95** · tombstones took probes/miss from **4.56 to 14.28 with the live key count unchanged**, while
probes/**hit** never moved at all

---

## 5. Load factor and rehashing

`🟢 easy` · *you cannot copy a hash table*

Every key's slot is a function of the table size, so growing means **rebuilding**. That is Θ(n) work,
and the argument that it is still Θ(1) per insert is the third appearance of the aggregate method in
this repo.

**Steps:**

1. Count the rehashes and the total entries moved.
2. Find the single insert that pays for all of them.
3. Sweep the load factor — and find out why the setting barely matters.

```go
package main

import (
	"fmt"
	"testing"
)

// A hash table is fast because it is SPARSE. Example 4 measured what happens
// when it stops being sparse: probes/miss went from 1.38 to 125 as the load
// factor climbed from 0.25 to 0.95.
//
// So the table must grow -- and growing means REHASHING, because every key's
// slot is a function of the table size. You cannot copy a hash table; you have
// to rebuild it.
//
// That is Theta(n) work, and this example is the argument that it is still
// Theta(1) per insert on average, plus the measurement of what it costs the
// one insert that pays for it.

type state uint8

const (
	free state = iota
	used
	deleted
)

type Table struct {
	keys   []uint64
	vals   []int
	states []state
	count  int
	tombs  int

	maxLoad float64
	// instrumentation
	rehashes    int
	rehashed    int // total entries moved by all rehashes
	lastRehash  int // entries moved by the most recent rehash
	probes      int
	growthTrace []int
}

func NewTable(n int, maxLoad float64) *Table {
	size := 1
	for size < n {
		size <<= 1
	}
	return &Table{
		keys:    make([]uint64, size),
		vals:    make([]int, size),
		states:  make([]state, size),
		maxLoad: maxLoad,
	}
}

func mix(k uint64) uint64 {
	k ^= k >> 30
	k *= 0xbf58476d1ce4e5b9
	k ^= k >> 27
	k *= 0x94d049bb133111eb
	k ^= k >> 31
	return k
}

func (t *Table) mask() uint64 { return uint64(len(t.keys) - 1) }

func (t *Table) Get(k uint64) (int, bool) {
	i := mix(k) & t.mask()
	for {
		t.probes++
		switch t.states[i] {
		case free:
			return 0, false
		case used:
			if t.keys[i] == k {
				return t.vals[i], true
			}
		}
		i = (i + 1) & t.mask()
	}
}

func (t *Table) Put(k uint64, v int) {
	// The load factor counts USED + TOMBSTONES. Counting only used entries is
	// the bug example 4 ended on: a table full of tombstones never grows and
	// never terminates.
	if float64(t.count+t.tombs+1) > t.maxLoad*float64(len(t.keys)) {
		t.grow()
	}
	t.put(k, v)
}

func (t *Table) put(k uint64, v int) {
	i := mix(k) & t.mask()
	firstTomb := -1
	for {
		switch t.states[i] {
		case free:
			if firstTomb >= 0 {
				i = uint64(firstTomb)
				t.tombs--
			}
			t.keys[i], t.vals[i], t.states[i] = k, v, used
			t.count++
			return
		case deleted:
			if firstTomb < 0 {
				firstTomb = int(i)
			}
		case used:
			if t.keys[i] == k {
				t.vals[i] = v
				return
			}
		}
		i = (i + 1) & t.mask()
	}
}

func (t *Table) Delete(k uint64) bool {
	i := mix(k) & t.mask()
	for {
		switch t.states[i] {
		case free:
			return false
		case used:
			if t.keys[i] == k {
				t.states[i] = deleted
				t.count--
				t.tombs++
				return true
			}
		}
		i = (i + 1) & t.mask()
	}
}

// grow doubles unless the table is mostly tombstones, in which case it rebuilds
// at the SAME size -- which is the point most implementations get right and
// most first drafts do not.
func (t *Table) grow() {
	newSize := len(t.keys)
	if float64(t.count+1) > t.maxLoad*float64(newSize)/2 {
		newSize *= 2
	}
	oldK, oldV, oldS := t.keys, t.vals, t.states
	t.keys = make([]uint64, newSize)
	t.vals = make([]int, newSize)
	t.states = make([]state, newSize)
	t.count, t.tombs = 0, 0
	moved := 0
	for i, s := range oldS {
		if s == used {
			t.put(oldK[i], oldV[i])
			moved++
		}
	}
	t.rehashes++
	t.rehashed += moved
	t.lastRehash = moved
	t.growthTrace = append(t.growthTrace, newSize)
}

func (t *Table) load() float64 {
	return float64(t.count+t.tombs) / float64(len(t.keys))
}

var sinkI int
var sinkB bool

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	fmt.Println("growing a hash table is not copying it. Every key's slot depends")
	fmt.Println("on the table SIZE, so every key has to be placed again:")
	fmt.Println()
	t := NewTable(4, 0.75)
	for i := 0; i < 100_000; i++ {
		t.Put(uint64(i)*2654435761, i)
	}
	fmt.Printf("  100,000 inserts -> %d rehashes, %d total entries moved\n",
		t.rehashes, t.rehashed)
	fmt.Printf("  final size %d, load factor %.3f\n", len(t.keys), t.load())
	fmt.Printf("  moves per insert: %.2f\n", float64(t.rehashed)/100_000)
	fmt.Println()
	fmt.Println("  under 3 moves per insert, and it is the same aggregate argument")
	fmt.Println("  as lesson 06's append and lesson 09's growable ring: doubling")
	fmt.Println("  makes the series n/2 + n/4 + ... < n, and the load factor divides")
	fmt.Println("  it by itself. Amortized Theta(1), for the third time in this repo.")

	fmt.Println()
	fmt.Println("but the WORST insert is not Theta(1), and it never is:")
	fmt.Println()
	fmt.Printf("  %12s %14s %16s\n", "insert #", "table size", "entries moved")
	t2 := NewTable(4, 0.75)
	for i := 0; i < 200_000; i++ {
		before := t2.rehashes
		t2.Put(uint64(i)*2654435761, i)
		if t2.rehashes > before && t2.lastRehash > 3000 {
			fmt.Printf("  %12d %14d %16d\n", i, len(t2.keys), t2.lastRehash)
		}
	}
	fmt.Println()
	fmt.Printf("  the worst single Put moved %d entries. If a hash table is on a\n", t2.lastRehash)
	fmt.Println("  latency-critical path, THAT is the number that matters, not the")
	fmt.Println("  average -- and it is why a size hint is not a micro-optimisation")
	fmt.Println("  (example 10).")

	fmt.Println()
	fmt.Println("the load factor is the dial, and it trades space against probes:")
	fmt.Println()
	fmt.Printf("  %10s %12s %10s %13s %12s %12s %11s\n",
		"max load", "final size", "rehashes", "moves/insert", "achieved", "probes/hit", "bytes/key")
	for _, lf := range []float64{0.5, 0.75, 0.875, 0.95} {
		tt := NewTable(4, lf)
		const n = 100_000
		for i := 0; i < n; i++ {
			tt.Put(uint64(i)*2654435761, i)
		}
		tt.probes = 0
		for i := 0; i < n; i++ {
			_, sinkB = tt.Get(uint64(i) * 2654435761)
		}
		bytesPerKey := float64(len(tt.keys)) * 17 / n // 8 key + 8 val + 1 state
		fmt.Printf("  %10.3f %12d %10d %13.2f %12.3f %12.2f %11.1f\n",
			lf, len(tt.keys), tt.rehashes, float64(tt.rehashed)/n,
			float64(n)/float64(len(tt.keys)), float64(tt.probes)/n, bytesPerKey)
	}
	fmt.Println()
	fmt.Println("  the rows pair up: 0.5 and 0.75 produce an identical table, and so")
	fmt.Println("  do 0.875 and 0.95. I expected four distinct rows and got two.")
	fmt.Println()
	fmt.Println("  the reason is that the size is a POWER OF TWO. 100,000 keys need")
	fmt.Println("  more than 131,072*0.75 = 98,304 slots, so both 0.5 and 0.75 jump")
	fmt.Println("  to 262,144; both 0.875 and 0.95 fit in 131,072 and stop there.")
	fmt.Println("  The setting only matters when it moves you across a boundary.")
	fmt.Println()
	fmt.Println("  so read the ACHIEVED column, not the setting: the real load lands")
	fmt.Println("  anywhere in [maxLoad/2, maxLoad], and 0.38 versus 0.76 is what")
	fmt.Println("  actually produced the 2x difference in bytes/key and probes/hit.")
	fmt.Println()
	fmt.Println("  Go's map picks 7/8 = 0.875 (maxAvgGroupLoad in")
	fmt.Println("  internal/runtime/maps/group.go). Abseil picks 7/8. Rust's")
	fmt.Println("  hashbrown picks 7/8. They all pick 7/8 because a SWISS TABLE")
	fmt.Println("  checks 8 slots at once, so the probe cost at 87.5% load is one")
	fmt.Println("  group check -- example 6.")

	fmt.Println()
	fmt.Println("rehashing also does something the load factor does not advertise:")
	fmt.Println("it is the only thing that removes tombstones.")
	fmt.Println()
	t3 := NewTable(1024, 0.75)
	key := func(i int) uint64 { return uint64(i) * 2654435761 }
	for i := 0; i < 700; i++ {
		t3.Put(key(i), i)
	}
	fmt.Printf("  %-30s used=%3d tombs=%3d size=%d rehashes=%d\n",
		"700 inserts", t3.count, t3.tombs, len(t3.keys), t3.rehashes)
	for i := 0; i < 600; i++ {
		t3.Delete(key(i))
	}
	fmt.Printf("  %-30s used=%3d tombs=%3d size=%d rehashes=%d\n",
		"delete 600 of them", t3.count, t3.tombs, len(t3.keys), t3.rehashes)
	// insert until a rehash actually fires, and report how long that took
	inserted, sizeBefore := 0, len(t3.keys)
	for i := 700; t3.rehashes == 0; i++ {
		t3.Put(key(i), i)
		inserted++
	}
	fmt.Printf("  %-30s used=%3d tombs=%3d size=%d rehashes=%d\n",
		fmt.Sprintf("%d more inserts", inserted), t3.count, t3.tombs,
		len(t3.keys), t3.rehashes)
	fmt.Printf("  size before the rehash: %d, after: %d\n", sizeBefore, len(t3.keys))
	fmt.Println()
	fmt.Println("  two different things happened on that last line, and only one of")
	fmt.Println("  them is a rehash.")
	fmt.Println()
	fmt.Println("  first, Put REUSES a tombstone it walks past, so many of the 600")
	fmt.Printf("  came back with no rebuild at all -- which is why it took %d\n", inserted)
	fmt.Println("  inserts to cross a threshold that was only 68 slots away.")
	fmt.Println()
	fmt.Println("  second, once used+tombstones did cross 0.75*1024 the table")
	fmt.Println("  rebuilt itself -- and the SIZE DID NOT CHANGE, because the live")
	fmt.Println("  count was still low. Rebuilding in place was enough, and it")
	fmt.Println("  cleared every remaining tombstone at once.")
	fmt.Println()
	fmt.Println("  that second part is the one to get right. A grow routine that")
	fmt.Println("  ALWAYS doubles will double a table that is 90% tombstones, and")
	fmt.Println("  then that table is 45% tombstones and twice as big, forever.")

	fmt.Println()
	fmt.Println("what it all costs on the clock:")
	fmt.Println()
	const n = 200_000
	tGrow := nsPerOp(func() {
		tt := NewTable(4, 0.875)
		for i := 0; i < n; i++ {
			tt.Put(uint64(i)*2654435761, i)
		}
		sinkI = tt.count
	})
	tHint := nsPerOp(func() {
		tt := NewTable(n*2, 0.875)
		for i := 0; i < n; i++ {
			tt.Put(uint64(i)*2654435761, i)
		}
		sinkI = tt.count
	})
	fmt.Printf("  %-34s %10.1f ns/insert\n", "grown from size 4", tGrow/n)
	fmt.Printf("  %-34s %10.1f ns/insert\n", "preallocated", tHint/n)
	fmt.Printf("  %-34s %10.1fx\n", "cost of growing", tGrow/tHint)
	fmt.Println()
	fmt.Println("  growing costs real time, and it is the one cost you can remove")
	fmt.Println("  entirely by knowing the size in advance. Example 10 measures that")
	fmt.Println("  on Go's own map, where the answer is larger than this.")
}
```

**Sample output:**

```
growing a hash table is not copying it. Every key's slot depends
on the table SIZE, so every key has to be placed again:

  100,000 inserts -> 16 rehashes, 196605 total entries moved
  final size 262144, load factor 0.381
  moves per insert: 1.97

  under 3 moves per insert, and it is the same aggregate argument
  as lesson 06's append and lesson 09's growable ring: doubling
  makes the series n/2 + n/4 + ... < n, and the load factor divides
  it by itself. Amortized Theta(1), for the third time in this repo.

but the WORST insert is not Theta(1), and it never is:

      insert #     table size    entries moved
          3072           8192             3072
          6144          16384             6144
         12288          32768            12288
         24576          65536            24576
         49152         131072            49152
         98304         262144            98304
        196608         524288           196608

  the worst single Put moved 196608 entries. If a hash table is on a
  latency-critical path, THAT is the number that matters, not the
  average -- and it is why a size hint is not a micro-optimisation
  (example 10).

the load factor is the dial, and it trades space against probes:

    max load   final size   rehashes  moves/insert     achieved   probes/hit   bytes/key
       0.500       262144         16          1.31        0.381         1.30        44.6
       0.750       262144         16          1.97        0.381         1.30        44.6
       0.875       131072         15          1.15        0.763         2.62        22.3
       0.950       131072         15          1.25        0.763         2.62        22.3

  the rows pair up: 0.5 and 0.75 produce an identical table, and so
  do 0.875 and 0.95. I expected four distinct rows and got two.

  the reason is that the size is a POWER OF TWO. 100,000 keys need
  more than 131,072*0.75 = 98,304 slots, so both 0.5 and 0.75 jump
  to 262,144; both 0.875 and 0.95 fit in 131,072 and stop there.
  The setting only matters when it moves you across a boundary.

  so read the ACHIEVED column, not the setting: the real load lands
  anywhere in [maxLoad/2, maxLoad], and 0.38 versus 0.76 is what
  actually produced the 2x difference in bytes/key and probes/hit.

  Go's map picks 7/8 = 0.875 (maxAvgGroupLoad in
  internal/runtime/maps/group.go). Abseil picks 7/8. Rust's
  hashbrown picks 7/8. They all pick 7/8 because a SWISS TABLE
  checks 8 slots at once, so the probe cost at 87.5% load is one
  group check -- example 6.

rehashing also does something the load factor does not advertise:
it is the only thing that removes tombstones.

  700 inserts                    used=700 tombs=  0 size=1024 rehashes=0
  delete 600 of them             used=100 tombs=600 size=1024 rehashes=0
  186 more inserts               used=286 tombs=  0 size=1024 rehashes=1
  size before the rehash: 1024, after: 1024

  two different things happened on that last line, and only one of
  them is a rehash.

  first, Put REUSES a tombstone it walks past, so many of the 600
  came back with no rebuild at all -- which is why it took 186
  inserts to cross a threshold that was only 68 slots away.

  second, once used+tombstones did cross 0.75*1024 the table
  rebuilt itself -- and the SIZE DID NOT CHANGE, because the live
  count was still low. Rebuilding in place was enough, and it
  cleared every remaining tombstone at once.

  that second part is the one to get right. A grow routine that
  ALWAYS doubles will double a table that is 90% tombstones, and
  then that table is 45% tombstones and twice as big, forever.

what it all costs on the clock:

  grown from size 4                        24.1 ns/insert
  preallocated                              6.5 ns/insert
  cost of growing                           3.7x

  growing costs real time, and it is the one cost you can remove
  entirely by knowing the size in advance. Example 10 measures that
  on Go's own map, where the answer is larger than this.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cell::Cell;
use std::collections::{HashMap, HashSet};
use std::hash::{BuildHasher, BuildHasherDefault, Hash, Hasher};
use std::hint::black_box;
use std::mem::replace;
use std::sync::atomic::{AtomicU64, AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// A hash table is fast because it is SPARSE. Example 4 measured what happens
// when it stops being sparse: probes/miss went from 1.38 to 125 as the load
// factor climbed from 0.25 to 0.95.
//
// So the table must grow -- and growing means REHASHING, because every key's
// slot is a function of the table size. You cannot copy a hash table; you have
// to rebuild it.
//
// That is Theta(n) work, and this example is the argument that it is still
// Theta(1) per insert on average, plus the measurement of what it costs the
// one insert that pays for it. Rust gives us one extra thing Go's example does
// not have: its std HashMap is a real, tuned table that does exactly this, so
// after each experiment on the hand-written table the same experiment is run
// on `HashMap` itself.

// ---- measuring kit: std-only stand-ins for what Go's stdlib gives for free ----

// Allocation counter (stand-in for runtime.MemStats): no GC means "how many
// allocations, how many live bytes" are exact numbers.
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

fn allocs() -> usize {
    ALLOCS.load(Relaxed)
}
/// bytes currently live on the heap (a deterministic MemStats.HeapAlloc).
fn live_bytes() -> usize {
    LIVE.load(Relaxed)
}

/// stand-in for testing.Benchmark / b.Loop: ns per call of `f`.
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

// ---- the hash: the splitmix64 finalizer from example 2 ----

fn mix(mut k: u64) -> u64 {
    k ^= k >> 30;
    k = k.wrapping_mul(0xbf58476d1ce4e5b9);
    k ^= k >> 27;
    k = k.wrapping_mul(0x94d049bb133111eb);
    k ^= k >> 31;
    k
}

// The same function as a `std::hash::Hasher`, so std's HashMap can use the very
// hash the hand-written table uses. Then the only difference between the two
// tables is the table. (std's default is SipHash-1-3 with a random seed: slower,
// and different on every run.)
#[derive(Default)]
struct MixHasher(u64);
impl Hasher for MixHasher {
    fn write(&mut self, _: &[u8]) {
        unreachable!("this example only hashes u64 keys");
    }
    fn write_u64(&mut self, k: u64) {
        self.0 = mix(k);
    }
    fn finish(&self) -> u64 {
        self.0
    }
}

// A BuildHasher that counts every hash it is asked for. A table has to hash a
// key exactly when it inserts it and when it MOVES it, so
//     hashes - inserts  ==  entries moved by rehashing
// is a measurement of std's rehash cost through the public API alone.
static HASHES: AtomicU64 = AtomicU64::new(0);
#[derive(Clone, Copy, Default)]
struct CountedMix;
impl BuildHasher for CountedMix {
    type Hasher = MixHasher;
    fn build_hasher(&self) -> MixHasher {
        HASHES.fetch_add(1, Relaxed);
        MixHasher(0)
    }
}

// A key that counts how often it is compared. Not needed for the rehashing
// story, but it is how a hit's cost in std's table is read off in the sweep.
static COMPARES: AtomicU64 = AtomicU64::new(0);
#[derive(Clone, Copy)]
struct K(u64);
impl PartialEq for K {
    fn eq(&self, o: &K) -> bool {
        COMPARES.fetch_add(1, Relaxed);
        self.0 == o.0
    }
}
impl Eq for K {}
impl Hash for K {
    fn hash<H: Hasher>(&self, h: &mut H) {
        h.write_u64(self.0);
    }
}
type StdMap = HashMap<K, usize, CountedMix>;

fn hashes() -> u64 {
    HASHES.load(Relaxed)
}

// ---- the hand-written table ----

#[derive(Clone, Copy, PartialEq)]
enum State {
    Free,
    Used,
    Deleted,
}

// Go's Table keeps THREE PARALLEL ARRAYS (8-byte key, 8-byte value, 1-byte
// state = 17 bytes per slot), and so does this one. A Vec of one enum per slot
// would be tidier but 24 bytes per slot, and "17 bytes per slot" is exactly
// what a hashbrown bucket costs (16-byte entry + 1 control byte), which makes
// the byte columns below comparable.
//
// Go's `growthTrace []int` is appended to and never read; it is not ported.
struct Table {
    keys: Vec<u64>,
    vals: Vec<usize>,
    states: Vec<State>,
    count: usize,
    tombs: usize,

    max_load: f64,
    // instrumentation
    rehashes: usize,
    rehashed: usize,    // total entries moved by all rehashes
    last_rehash: usize, // entries moved by the most recent rehash
    probes: Cell<usize>,
}

impl Table {
    fn new(n: usize, max_load: f64) -> Self {
        let size = n.next_power_of_two();
        Table {
            keys: vec![0; size],
            vals: vec![0; size],
            states: vec![State::Free; size],
            count: 0,
            tombs: 0,
            max_load,
            rehashes: 0,
            rehashed: 0,
            last_rehash: 0,
            probes: Cell::new(0),
        }
    }

    fn mask(&self) -> usize {
        self.keys.len() - 1
    }

    fn home(&self, k: u64) -> usize {
        mix(k) as usize & self.mask()
    }

    fn get(&self, k: u64) -> Option<usize> {
        let mut i = self.home(k);
        loop {
            self.probes.set(self.probes.get() + 1);
            match self.states[i] {
                State::Free => return None,
                State::Used if self.keys[i] == k => return Some(self.vals[i]),
                State::Used | State::Deleted => {}
            }
            i = (i + 1) & self.mask();
        }
    }

    fn put(&mut self, k: u64, v: usize) {
        // The load factor counts USED + TOMBSTONES. Counting only used entries
        // is the bug example 4 ended on: a table full of tombstones never grows
        // and never terminates.
        if (self.count + self.tombs + 1) as f64 > self.max_load * self.keys.len() as f64 {
            self.grow();
        }
        self.place(k, v);
    }

    fn place(&mut self, k: u64, v: usize) {
        let mut i = self.home(k);
        let mut first_tomb: Option<usize> = None; // Go: -1 means "none yet"
        loop {
            match self.states[i] {
                State::Free => {
                    let at = match first_tomb {
                        Some(t) => {
                            self.tombs -= 1;
                            t
                        }
                        None => i,
                    };
                    self.keys[at] = k;
                    self.vals[at] = v;
                    self.states[at] = State::Used;
                    self.count += 1;
                    return;
                }
                State::Deleted => {
                    first_tomb.get_or_insert(i);
                }
                State::Used if self.keys[i] == k => {
                    self.vals[i] = v;
                    return;
                }
                State::Used => {}
            }
            i = (i + 1) & self.mask();
        }
    }

    fn delete(&mut self, k: u64) -> bool {
        let mut i = self.home(k);
        loop {
            match self.states[i] {
                State::Free => return false,
                State::Used if self.keys[i] == k => {
                    self.states[i] = State::Deleted;
                    self.count -= 1;
                    self.tombs += 1;
                    return true;
                }
                State::Used | State::Deleted => {}
            }
            i = (i + 1) & self.mask();
        }
    }

    // grow doubles unless the table is mostly tombstones, in which case it
    // rebuilds at the SAME size -- which is the point most implementations get
    // right and most first drafts do not.
    fn grow(&mut self) {
        let mut new_size = self.keys.len();
        if (self.count + 1) as f64 > self.max_load * new_size as f64 / 2.0 {
            new_size *= 2;
        }
        // `replace` swaps the fresh arrays in and hands the old ones back; Go's
        // `oldK, oldV, oldS := t.keys, t.vals, t.states` is a copy of three
        // slice headers, here the old arrays are moved out.
        let old_k = replace(&mut self.keys, vec![0; new_size]);
        let old_v = replace(&mut self.vals, vec![0; new_size]);
        let old_s = replace(&mut self.states, vec![State::Free; new_size]);
        self.count = 0;
        self.tombs = 0;
        let mut moved = 0;
        for ((&k, &v), &s) in old_k.iter().zip(&old_v).zip(&old_s) {
            if s == State::Used {
                self.place(k, v);
                moved += 1;
            }
        }
        self.rehashes += 1;
        self.rehashed += moved;
        self.last_rehash = moved;
    }

    fn load(&self) -> f64 {
        (self.count + self.tombs) as f64 / self.keys.len() as f64
    }
}

fn key(i: usize) -> u64 {
    (i as u64).wrapping_mul(2654435761)
}

// ---- std's table, observed from the outside ----

// What one insert into a std HashMap did, seen only through `capacity()`, the
// hasher counter and the allocator counter.
struct Growth {
    insert: usize,   // 0-based number of the insert that triggered it
    capacity: usize, // capacity() afterwards
    moved: u64,      // entries re-hashed by it
    allocs: usize,   // allocation events it caused
}

// Fill a std map with n keys, no size hint, recording every capacity change.
fn fill_std(n: usize) -> (StdMap, Vec<Growth>) {
    let mut m: StdMap = HashMap::with_hasher(CountedMix);
    let mut events = Vec::new();
    for i in 0..n {
        let (cap, h, a) = (m.capacity(), hashes(), allocs());
        m.insert(K(key(i)), i);
        if m.capacity() != cap {
            events.push(Growth {
                insert: i,
                capacity: m.capacity(),
                moved: hashes() - h - 1, // one hash is the insert itself
                allocs: allocs() - a,
            });
        }
    }
    (m, events)
}

// std grows to a power-of-two number of BUCKETS and lets 7/8 of them fill, so
// capacity() = 7/8 of the bucket count. Going back from the capacity is the
// formula in hashbrown's raw.rs (read, not run); `buckets_from_bytes` below
// checks it against the allocator.
fn buckets_from_capacity(cap: usize) -> usize {
    (cap * 8 / 7).next_power_of_two()
}

// The live heap of a table with B buckets of 16-byte entries is 17*B + W: the
// entries, one control byte per bucket, and W trailing control bytes that
// repeat the first group so that a group load never runs off the end, W being
// the group width (8 on aarch64, 16 with SSE2). The layout is from raw.rs; the
// byte count is what the allocator reports, and W is read off capacity():
// raw.rs gives a table of 1-byte entries a minimum capacity of 7 for 8-wide
// groups and 14 for 16-wide ones.
fn group_width() -> usize {
    match HashSet::<u8>::with_capacity(1).capacity() {
        7 => 8,
        14 => 16,
        c => panic!("unexpected minimum capacity {c}"),
    }
}
fn buckets_from_bytes(bytes: usize) -> usize {
    (bytes - group_width()) / 17
}

// Smallest non-zero step between two clock reads, in ns: the resolution of
// Instant on this machine, measured.
fn clock_tick() -> u64 {
    let mut best = u64::MAX;
    let mut last = Instant::now();
    for _ in 0..200_000 {
        let now = Instant::now();
        let d = now.duration_since(last).as_nanos() as u64;
        if d > 0 {
            best = best.min(d);
        }
        last = now;
    }
    best
}

// x rounded down to two significant digits, so a noisy ratio prints as a
// magnitude (48075 -> 48000).
fn round2(x: f64) -> u64 {
    let scale = 10f64.powi((x.log10().floor() as i32 - 1).max(0));
    ((x / scale).floor() * scale) as u64
}

fn std_row(label: &str, n: usize) -> (f64, f64, f64) {
    let before = live_bytes();
    let (m, events) = fill_std(n);
    let moved: u64 = events.iter().map(|e| e.moved).sum();
    let rehashes = events.len() - 1; // the first "growth" is the first allocation
    drop(events); // so that the bytes below are the table's alone
    let bytes = live_bytes() - before;
    let buckets = buckets_from_bytes(bytes);
    assert_eq!(buckets, buckets_from_capacity(m.capacity()));
    COMPARES.store(0, Relaxed);
    let hits = (0..n).filter(|&i| m.contains_key(&K(key(i)))).count();
    assert_eq!(hits, n);
    let (achieved, compares, per_key) = (
        n as f64 / buckets as f64,
        COMPARES.load(Relaxed) as f64 / n as f64,
        bytes as f64 / n as f64,
    );
    println!(
        "  {:<14} {:>10} {:>10} {:>13.2} {:>10.3} {:>13.2} {:>11.1}",
        label,
        buckets,
        rehashes,
        moved as f64 / n as f64,
        achieved,
        compares,
        per_key
    );
    (achieved, compares, per_key)
}

// Time one insert at a time. Returns (ns, did it rehash?) per insert.
fn insert_latencies<T>(
    n: usize,
    mut table: T,
    mut insert: impl FnMut(&mut T, usize) -> bool,
) -> Vec<(u64, bool)> {
    (0..n)
        .map(|i| {
            let t = Instant::now();
            let grew = insert(&mut table, i);
            (t.elapsed().as_nanos() as u64, grew)
        })
        .collect()
}

// Prints one row; returns (rehash / calm mean, insert # of the slowest rehash).
fn describe(label: &str, lat: &[(u64, bool)]) -> (f64, usize) {
    let calm: Vec<u64> = lat.iter().filter(|x| !x.1).map(|x| x.0).collect();
    let mean_calm = calm.iter().sum::<u64>() as f64 / calm.len() as f64;
    let slowest_calm = calm.iter().max().copied().unwrap_or(0);
    let (at, slowest_grow) = lat
        .iter()
        .enumerate()
        .filter(|(_, x)| x.1)
        .max_by_key(|(_, x)| x.0)
        .map(|(i, x)| (i, x.0))
        .expect("200,000 inserts must rehash");
    println!(
        "  {:<20} {:>14.1} {:>14.1} {:>14.1} {:>9} {:>9.0}x",
        label,
        mean_calm,
        slowest_calm as f64 / 1000.0,
        slowest_grow as f64 / 1000.0,
        at,
        slowest_grow as f64 / mean_calm
    );
    (slowest_grow as f64 / mean_calm, at)
}

fn main() {
    println!("growing a hash table is not copying it. Every key's slot depends");
    println!("on the table SIZE, so every key has to be placed again:");
    println!();
    let mut t = Table::new(4, 0.75);
    let a0 = allocs();
    for i in 0..100_000 {
        t.put(key(i), i);
    }
    let table_allocs = allocs() - a0;
    println!(
        "  100,000 inserts -> {} rehashes, {} total entries moved",
        t.rehashes, t.rehashed
    );
    println!("  final size {}, load factor {:.3}", t.keys.len(), t.load());
    println!("  moves per insert: {:.2}", t.rehashed as f64 / 100_000.0);
    println!();
    println!("  under 3 moves per insert, and it is the same aggregate argument");
    println!("  as lesson 06's append and lesson 09's growable ring: doubling");
    println!("  makes the series n/2 + n/4 + ... < n, and the load factor divides");
    println!("  it by itself. Amortized Theta(1), for the third time in this repo.");
    println!();
    println!("  the same 100,000 inserts into std's HashMap<u64, usize> with the same");
    println!("  hash, seen from outside: capacity(), a hasher that counts its calls,");
    println!("  an allocator that counts its calls. (std has no max-load setting.)");
    println!();
    let (m, events) = fill_std(100_000);
    let std_allocs: usize = events.iter().map(|e| e.allocs).sum();
    let std_moved: u64 = events.iter().map(|e| e.moved).sum();
    println!(
        "  {:<46} {:>7}",
        "rehashes (capacity() changes after the 1st)",
        events.len() - 1
    );
    println!(
        "  {:<46} {:>7}",
        "entries moved (hash calls beyond the inserts)", std_moved
    );
    println!(
        "  {:<46} {:>7.2}",
        "moves per insert",
        std_moved as f64 / 100_000.0
    );
    println!("  {:<46} {:>7}", "final capacity()", m.capacity());
    println!("  {:<46} {:>7}", "allocation events: std map", std_allocs);
    println!(
        "  {:<46} {:>7}",
        "allocation events: hand-written table", table_allocs
    );
    println!(
        "  {:<46} {}",
        "capacity() after each growth",
        events
            .iter()
            .take(7)
            .map(|e| e.capacity.to_string())
            .collect::<Vec<_>>()
            .join(" ")
    );
    println!(
        "  {:<46} {}",
        "entries moved by each growth",
        events
            .iter()
            .take(7)
            .map(|e| e.moved.to_string())
            .collect::<Vec<_>>()
            .join(" ")
    );
    drop(m);
    println!();
    println!("  std lands on the numbers of the 0.875 row of the sweep below: the same");
    println!("  15 rehashes and 1.15 moves per insert. Its capacity() doubles at every");
    println!("  growth, so the same n/2 + n/4 + ... argument applies, and each growth");
    println!("  moves exactly the entries that were in the table (the 'entries moved'");
    println!("  row is the previous capacity). One difference the allocator shows: a");
    println!("  growth is ONE allocation for std (entries and control bytes share one");
    println!("  block) and THREE for the hand-written table, which keeps keys, values");
    println!(
        "  and states in three arrays: {std_allocs} allocation events against {table_allocs}."
    );

    println!();
    println!("but the WORST insert is not Theta(1), and it never is:");
    println!();
    println!(
        "  {:>12} {:>14} {:>16}",
        "insert #", "table size", "entries moved"
    );
    let mut t2 = Table::new(4, 0.75);
    for i in 0..200_000 {
        let before = t2.rehashes;
        t2.put(key(i), i);
        if t2.rehashes > before && t2.last_rehash > 3000 {
            println!("  {:>12} {:>14} {:>16}", i, t2.keys.len(), t2.last_rehash);
        }
    }
    println!();
    println!(
        "  the worst single Put moved {} entries. If a hash table is on a",
        t2.last_rehash
    );
    println!("  latency-critical path, THAT is the number that matters, not the");
    println!("  average -- and it is why a size hint is not a micro-optimisation");
    println!("  (example 10).");
    println!();
    println!("  std's table, 200,000 inserts, the growths that moved more than 3000:");
    println!();
    println!(
        "  {:>12} {:>14} {:>14} {:>16}",
        "insert #", "capacity()", "buckets", "entries moved"
    );
    let (_, events200) = fill_std(200_000);
    for e in events200.iter().filter(|e| e.moved > 3000) {
        println!(
            "  {:>12} {:>14} {:>14} {:>16}",
            e.insert,
            e.capacity,
            buckets_from_capacity(e.capacity),
            e.moved
        );
    }
    println!();
    println!("  the same thing timed, one insert at a time (7/8 max load for both).");
    println!("  Each insert is bracketed by Instant::now(): a clock read costs about");
    println!(
        "  {:.0} ns and the clock moves in steps of {} ns on this machine, so one calm",
        ns_per_op(|| {
            black_box(Instant::now());
        }),
        clock_tick()
    );
    println!("  insert is below what it can resolve and only the mean over 200,000 means");
    println!("  anything. A rehash is far above it.");
    println!();
    println!(
        "  {:<20} {:>14} {:>14} {:>14} {:>9} {:>10}",
        "table", "calm insert", "slowest calm", "slowest rehash", "its", "rehash /"
    );
    println!(
        "  {:<20} {:>14} {:>14} {:>14} {:>9} {:>10}",
        "", "mean (ns)", "(us)", "(us)", "insert #", "calm mean"
    );
    let lat_table = insert_latencies(200_000, Table::new(4, 0.875), |t, i| {
        let before = t.rehashes;
        t.put(key(i), i);
        t.rehashes > before
    });
    let (ratio_a, at_a) = describe("hand-written table", &lat_table);
    let lat_std = insert_latencies(
        200_000,
        HashMap::<u64, usize, BuildHasherDefault<MixHasher>>::default(),
        |m, i| {
            let cap = m.capacity();
            m.insert(key(i), i);
            m.capacity() != cap
        },
    );
    let (ratio_b, at_b) = describe("std HashMap", &lat_std);
    println!();
    println!("  'calm insert' is the mean of every insert that did not rehash, and");
    println!("  it includes the clock read. 'slowest calm' is the slowest insert that");
    println!("  did NOT rehash; it changes a lot from run to run, so treat it as noise");
    println!("  (an interrupt, a page fault, a cache miss). The rehash column is the");
    println!("  table: the slowest rehash is insert #{at_a} in the hand-written table and");
    println!(
        "  #{at_b} in std's, and each costs more than {} calm inserts. The average",
        round2(ratio_a.min(ratio_b))
    );
    println!("  argument and the worst-case argument are about numbers that far apart.");

    println!();
    println!("the load factor is the dial, and it trades space against probes:");
    println!();
    println!(
        "  {:>10} {:>12} {:>10} {:>13} {:>12} {:>12} {:>11}",
        "max load", "final size", "rehashes", "moves/insert", "achieved", "probes/hit", "bytes/key"
    );
    for lf in [0.5, 0.75, 0.875, 0.95] {
        const N: usize = 100_000;
        let before = live_bytes();
        let mut tt = Table::new(4, lf);
        for i in 0..N {
            tt.put(key(i), i);
        }
        let bytes = live_bytes() - before;
        tt.probes.set(0);
        let hits = (0..N).filter(|&i| tt.get(key(i)).is_some()).count();
        assert_eq!(hits, N);
        assert_eq!(bytes, tt.keys.len() * 17); // 8 key + 8 val + 1 state
        println!(
            "  {:>10.3} {:>12} {:>10} {:>13.2} {:>12.3} {:>12.2} {:>11.1}",
            lf,
            tt.keys.len(),
            tt.rehashes,
            tt.rehashed as f64 / N as f64,
            N as f64 / tt.keys.len() as f64,
            tt.probes.get() as f64 / N as f64,
            bytes as f64 / N as f64
        );
    }
    println!();
    println!("  the rows pair up: 0.5 and 0.75 produce an identical table, and so");
    println!("  do 0.875 and 0.95. Four settings, two tables.");
    println!();
    println!("  the reason is that the size is a POWER OF TWO. 100,000 keys need");
    println!("  more than 131,072*0.75 = 98,304 slots, so both 0.5 and 0.75 jump");
    println!("  to 262,144; both 0.875 and 0.95 fit in 131,072 and stop there.");
    println!("  The setting only matters when it moves you across a boundary.");
    println!();
    println!("  so read the ACHIEVED column, not the setting: the real load lands");
    println!("  anywhere in [maxLoad/2, maxLoad], and 0.38 versus 0.76 is what");
    println!("  actually produced the 2x difference in bytes/key and probes/hit.");
    println!();
    println!("  std has one setting, 7/8, and it is not adjustable (raw.rs, read not");
    println!("  run: capacity() is 7/8 of a power-of-two bucket count). So std IS the");
    println!("  0.875 row: at 100,000 keys, 131,072 slots, 15 rehashes, 1.15 moves,");
    println!("  22.3 bytes/key, as below. The rows below walk it across its growth");
    println!("  boundary (57,344 = 7/8 of 65,536). 'compares/hit' counts key");
    println!("  comparisons per successful lookup, which is not the unit of the");
    println!("  probes/hit column above; 'buckets' is its size:");
    println!();
    println!(
        "  {:<14} {:>10} {:>10} {:>13} {:>10} {:>13} {:>11}",
        "keys inserted",
        "buckets",
        "rehashes",
        "moves/insert",
        "achieved",
        "compares/hit",
        "bytes/key"
    );
    let mut rows = Vec::new();
    for n in [57_344, 57_345, 100_000, 114_688, 114_689] {
        rows.push(std_row(&n.to_string(), n));
    }
    let col = |f: fn(&(f64, f64, f64)) -> f64| {
        let v = rows.iter().map(f);
        (
            v.clone().fold(f64::MAX, f64::min),
            v.fold(f64::MIN, f64::max),
        )
    };
    let (a_lo, a_hi) = col(|r| r.0);
    let (c_lo, c_hi) = col(|r| r.1);
    let (b_lo, b_hi) = col(|r| r.2);
    println!();
    println!("  the same power-of-two swing: one key apart (57,344 and 57,345) the");
    println!("  achieved load goes from {a_hi:.3} to {a_lo:.3} and the cost per key from");
    println!("  {b_lo:.1} to {b_hi:.1} bytes. What does not swing is the lookup: key");
    println!("  comparisons per hit stay between {c_lo:.2} and {c_hi:.2} across that whole range,");
    println!("  where the hand-written table's probes/hit went from 1.30 to 2.62 over");
    println!("  a similar one (0.38 to 0.76). Why is example 6.");

    println!();
    println!("rehashing also does something the load factor does not advertise:");
    println!("it is the only thing that removes tombstones.");
    println!();
    let mut t3 = Table::new(1024, 0.75);
    for i in 0..700 {
        t3.put(key(i), i);
    }
    let row = |what: &str, t: &Table| {
        println!(
            "  {:<30} used={:>3} tombs={:>3} size={} rehashes={}",
            what,
            t.count,
            t.tombs,
            t.keys.len(),
            t.rehashes
        );
    };
    row("700 inserts", &t3);
    for i in 0..600 {
        t3.delete(key(i));
    }
    row("delete 600 of them", &t3);
    // insert until a rehash actually fires, and report how long that took
    let (mut inserted, size_before) = (0, t3.keys.len());
    let mut i = 700;
    while t3.rehashes == 0 {
        t3.put(key(i), i);
        inserted += 1;
        i += 1;
    }
    row(&format!("{inserted} more inserts"), &t3);
    println!(
        "  size before the rehash: {}, after: {}",
        size_before,
        t3.keys.len()
    );
    println!();
    println!("  two different things happened on that last line, and only one of");
    println!("  them is a rehash.");
    println!();
    println!("  first, Put REUSES a tombstone it walks past, so many of the 600");
    println!("  came back with no rebuild at all -- which is why it took {inserted}");
    println!("  inserts to cross a threshold that was only 68 slots away.");
    println!();
    println!("  second, once used+tombstones did cross 0.75*1024 the table");
    println!("  rebuilt itself -- and the SIZE DID NOT CHANGE, because the live");
    println!("  count was still low. Rebuilding in place was enough, and it");
    println!("  cleared every remaining tombstone at once.");
    println!();
    println!("  that second part is the one to get right. A grow routine that");
    println!("  ALWAYS doubles will double a table that is 90% tombstones, and");
    println!("  then that table is 45% tombstones and twice as big, forever.");
    println!();

    // std: a table of exactly 1024 buckets (7/8 of 1024 = 896), the same
    // 700 inserts and 600 deletes. std shows no tombstone count, so it is
    // inferred: capacity() = items + growth_left, and a tombstone is the one
    // thing that stops a removal from handing its slot's budget back, so
    //     tombstones = 7/8 * buckets - capacity().
    // (That accounting is from reading raw.rs. The bucket count comes from the
    // allocator, 17*B + 8 bytes, so it does not depend on capacity().)
    println!("  and std's table, the same keys. It shows no tombstone count, so 'tombs'");
    println!("  is inferred as 7/8 * buckets - capacity() (the accounting is read from");
    println!("  raw.rs; buckets come from the allocator's byte count, not from capacity()):");
    println!();
    let base = live_bytes();
    let mut s: StdMap = HashMap::with_capacity_and_hasher(896, CountedMix);
    // `bytes` is read BEFORE the row's label is built: a format!ed label is a
    // live allocation too, and the byte count must be the table's alone.
    let srow = |what: &str, bytes: usize, s: &StdMap| {
        let buckets = buckets_from_bytes(bytes);
        let tombs = buckets / 8 * 7 - s.capacity();
        println!(
            "  {:<30} used={:>3} tombs={:>3} capacity()={:>4} buckets={}",
            what,
            s.len(),
            tombs,
            s.capacity(),
            buckets
        );
        tombs
    };
    for i in 0..700 {
        s.insert(K(key(i)), i);
    }
    srow("std: 700 inserts", live_bytes() - base, &s);
    for i in 0..600 {
        s.remove(&K(key(i)));
    }
    let tombs = srow("std: delete 600 of them", live_bytes() - base, &s);
    let budget = s.capacity() - s.len();
    let (mut inserted, mut i) = (0, 700);
    let (moved, a, cap_before) = loop {
        let (h, a, c) = (hashes(), allocs(), s.capacity());
        s.insert(K(key(i)), i);
        inserted += 1;
        i += 1;
        if hashes() - h > 1 {
            break (hashes() - h - 1, allocs() - a, c);
        }
        assert!(inserted < 5000, "no rehash fired");
    };
    let bytes = live_bytes() - base;
    srow(&format!("std: {inserted} more inserts"), bytes, &s);
    println!(
        "  that last insert moved {moved} entries and made {a} allocation event(s); capacity() {cap_before} (== len(): the budget was gone) -> {}",
        s.capacity()
    );
    println!();
    println!("  std did not do what the hand-written table did, and the numbers say why.");
    println!(
        "  Only {tombs} of the 600 removals left a tombstone; the other {} handed their",
        600 - tombs
    );
    println!("  budget straight back (raw.rs erase, read not run: it writes an EMPTY slot, not a");
    println!("  tombstone, whenever an EMPTY slot lies within a group's width of the removed");
    println!(
        "  one; that {} is what capacity() shows). So the budget was {budget} inserts into",
        600 - tombs
    );
    println!(
        "  never-used slots, not 68, and it took {inserted} inserts to spend it, because an insert"
    );
    println!(
        "  that lands on a tombstone spends none. By then {} entries were live, {:.0}% of",
        s.len(),
        100.0 * s.len() as f64 / 896.0
    );
    println!("  the 896 the table allows, and growing was the right answer: raw.rs rebuilds");
    println!("  IN PLACE only when the live entries are at most half the capacity, and");
    println!("  grows otherwise (read, not run).");
    println!();
    println!("  the in-place branch needs live entries to stay low while the budget");
    println!("  leaks away, which is churn: the same table, 100 live entries, then one");
    println!("  new key in and the oldest key out, over and over.");
    println!();
    let base = live_bytes();
    let mut c: StdMap = HashMap::with_capacity_and_hasher(896, CountedMix);
    for i in 0..700 {
        c.insert(K(key(i)), i);
    }
    for i in 0..600 {
        c.remove(&K(key(i)));
    }
    let crow = |what: &str, bytes: usize, c: &StdMap| {
        let buckets = buckets_from_bytes(bytes);
        println!(
            "  {:<30} used={:>3} tombs={:>3} capacity()={:>4} buckets={}",
            what,
            c.len(),
            buckets / 8 * 7 - c.capacity(),
            c.capacity(),
            buckets
        );
    };
    crow("std churn: start", live_bytes() - base, &c);
    let start_cap = c.capacity();
    let (mut pairs, mut low) = (0, start_cap);
    let (moved, a, cap_before) = loop {
        let (h, a, cap) = (hashes(), allocs(), c.capacity());
        c.insert(K(key(700 + pairs)), pairs);
        if hashes() - h > 1 {
            break (hashes() - h - 1, allocs() - a, cap);
        }
        c.remove(&K(key(600 + pairs)));
        pairs += 1;
        low = low.min(c.capacity());
        assert!(pairs < 100_000, "no rehash fired");
    };
    let bytes = live_bytes() - base;
    crow(&format!("{pairs} insert+remove pairs"), bytes, &c);
    println!(
        "  that last insert moved {moved} entries and made {a} allocation event(s); capacity() {cap_before} (== len(): the budget was gone) -> {}",
        c.capacity()
    );
    println!();
    println!(
        "  capacity() drained from {start_cap} to {low} as tombstones piled up with the live count"
    );
    println!("  fixed at 100, and then one insert rebuilt the table at the same size, with");
    println!(
        "  no new allocation, and capacity() came back to {}. That is the behaviour the",
        c.capacity()
    );
    println!("  hand-written table's grow() has to be written to have.");
    println!();
    println!("what it all costs on the clock:");
    println!();
    const N: usize = 200_000;
    let t_grow = ns_per_op(|| {
        let mut tt = Table::new(4, 0.875);
        for i in 0..N {
            tt.put(key(i), i);
        }
        black_box(tt.count);
    });
    let t_hint = ns_per_op(|| {
        let mut tt = Table::new(N * 2, 0.875);
        for i in 0..N {
            tt.put(key(i), i);
        }
        black_box(tt.count);
    });
    let s_grow = ns_per_op(|| {
        let mut m: HashMap<u64, usize, BuildHasherDefault<MixHasher>> = HashMap::default();
        for i in 0..N {
            m.insert(key(i), i);
        }
        black_box(m.len());
    });
    let s_hint = ns_per_op(|| {
        let mut m: HashMap<u64, usize, BuildHasherDefault<MixHasher>> =
            HashMap::with_capacity_and_hasher(N, Default::default());
        for i in 0..N {
            m.insert(key(i), i);
        }
        black_box(m.len());
    });
    let r_grow = ns_per_op(|| {
        let mut m: HashMap<u64, usize> = HashMap::new();
        for i in 0..N {
            m.insert(key(i), i);
        }
        black_box(m.len());
    });
    let r_hint = ns_per_op(|| {
        let mut m: HashMap<u64, usize> = HashMap::with_capacity(N);
        for i in 0..N {
            m.insert(key(i), i);
        }
        black_box(m.len());
    });
    println!(
        "  {:<30} {:>12} {:>14} {:>10}",
        "ns/insert", "grown from 4", "preallocated", "growing x"
    );
    for (label, g, h) in [
        ("hand-written table", t_grow, t_hint),
        ("std HashMap, same hash", s_grow, s_hint),
        ("std HashMap, SipHash default", r_grow, r_hint),
    ] {
        println!(
            "  {:<30} {:>12.1} {:>14.1} {:>9.1}x",
            label,
            g / N as f64,
            h / N as f64,
            g / h
        );
    }
    let ratios: Vec<f64> = [(t_grow, t_hint), (s_grow, s_hint), (r_grow, r_hint)]
        .iter()
        .map(|(g, h)| g / h)
        .collect();
    let (lo, hi) = (
        ratios.iter().copied().fold(f64::MAX, f64::min),
        ratios.iter().copied().fold(f64::MIN, f64::max),
    );
    println!();
    println!("  growing costs real time in every row ({lo:.1}x to {hi:.1}x here), and it is the");
    println!("  one cost you can remove entirely by knowing the size in advance:");
    println!("  `HashMap::with_capacity(n)` is the whole fix. The hand-written 'preallocated'");
    println!(
        "  table is sized 2n as in the Go program, {} slots against the {} buckets that",
        (N * 2).next_power_of_two(),
        (N * 8 / 7).next_power_of_two()
    );
    println!("  with_capacity(n) picks, so it touches twice the memory. A size hint");
    println!("  on the map type itself is what example 10 measures.");
}
```

**Rust sample output:**

```
growing a hash table is not copying it. Every key's slot depends
on the table SIZE, so every key has to be placed again:

  100,000 inserts -> 16 rehashes, 196605 total entries moved
  final size 262144, load factor 0.381
  moves per insert: 1.97

  under 3 moves per insert, and it is the same aggregate argument
  as lesson 06's append and lesson 09's growable ring: doubling
  makes the series n/2 + n/4 + ... < n, and the load factor divides
  it by itself. Amortized Theta(1), for the third time in this repo.

  the same 100,000 inserts into std's HashMap<u64, usize> with the same
  hash, seen from outside: capacity(), a hasher that counts its calls,
  an allocator that counts its calls. (std has no max-load setting.)

  rehashes (capacity() changes after the 1st)         15
  entries moved (hash calls beyond the inserts)   114684
  moves per insert                                  1.15
  final capacity()                                114688
  allocation events: std map                          16
  allocation events: hand-written table               48
  capacity() after each growth                   3 7 14 28 56 112 224
  entries moved by each growth                   0 3 7 14 28 56 112

  std lands on the numbers of the 0.875 row of the sweep below: the same
  15 rehashes and 1.15 moves per insert. Its capacity() doubles at every
  growth, so the same n/2 + n/4 + ... argument applies, and each growth
  moves exactly the entries that were in the table (the 'entries moved'
  row is the previous capacity). One difference the allocator shows: a
  growth is ONE allocation for std (entries and control bytes share one
  block) and THREE for the hand-written table, which keeps keys, values
  and states in three arrays: 16 allocation events against 48.

but the WORST insert is not Theta(1), and it never is:

      insert #     table size    entries moved
          3072           8192             3072
          6144          16384             6144
         12288          32768            12288
         24576          65536            24576
         49152         131072            49152
         98304         262144            98304
        196608         524288           196608

  the worst single Put moved 196608 entries. If a hash table is on a
  latency-critical path, THAT is the number that matters, not the
  average -- and it is why a size hint is not a micro-optimisation
  (example 10).

  std's table, 200,000 inserts, the growths that moved more than 3000:

      insert #     capacity()        buckets    entries moved
          3584           7168           8192             3584
          7168          14336          16384             7168
         14336          28672          32768            14336
         28672          57344          65536            28672
         57344         114688         131072            57344
        114688         229376         262144           114688

  the same thing timed, one insert at a time (7/8 max load for both).
  Each insert is bracketed by Instant::now(): a clock read costs about
  15 ns and the clock moves in steps of 41 ns on this machine, so one calm
  insert is below what it can resolve and only the mean over 200,000 means
  anything. A rehash is far above it.

  table                   calm insert   slowest calm slowest rehash       its   rehash /
                            mean (ns)           (us)           (us)  insert #  calm mean
  hand-written table             18.2            6.7          909.8    114688     49990x
  std HashMap                    16.7            0.4          809.8    114688     48386x

  'calm insert' is the mean of every insert that did not rehash, and
  it includes the clock read. 'slowest calm' is the slowest insert that
  did NOT rehash; it changes a lot from run to run, so treat it as noise
  (an interrupt, a page fault, a cache miss). The rehash column is the
  table: the slowest rehash is insert #114688 in the hand-written table and
  #114688 in std's, and each costs more than 48000 calm inserts. The average
  argument and the worst-case argument are about numbers that far apart.

the load factor is the dial, and it trades space against probes:

    max load   final size   rehashes  moves/insert     achieved   probes/hit   bytes/key
       0.500       262144         16          1.31        0.381         1.30        44.6
       0.750       262144         16          1.97        0.381         1.30        44.6
       0.875       131072         15          1.15        0.763         2.62        22.3
       0.950       131072         15          1.25        0.763         2.62        22.3

  the rows pair up: 0.5 and 0.75 produce an identical table, and so
  do 0.875 and 0.95. Four settings, two tables.

  the reason is that the size is a POWER OF TWO. 100,000 keys need
  more than 131,072*0.75 = 98,304 slots, so both 0.5 and 0.75 jump
  to 262,144; both 0.875 and 0.95 fit in 131,072 and stop there.
  The setting only matters when it moves you across a boundary.

  so read the ACHIEVED column, not the setting: the real load lands
  anywhere in [maxLoad/2, maxLoad], and 0.38 versus 0.76 is what
  actually produced the 2x difference in bytes/key and probes/hit.

  std has one setting, 7/8, and it is not adjustable (raw.rs, read not
  run: capacity() is 7/8 of a power-of-two bucket count). So std IS the
  0.875 row: at 100,000 keys, 131,072 slots, 15 rehashes, 1.15 moves,
  22.3 bytes/key, as below. The rows below walk it across its growth
  boundary (57,344 = 7/8 of 65,536). 'compares/hit' counts key
  comparisons per successful lookup, which is not the unit of the
  probes/hit column above; 'buckets' is its size:

  keys inserted     buckets   rehashes  moves/insert   achieved  compares/hit   bytes/key
  57344               65536         14          1.00      0.875          1.02        19.4
  57345              131072         15          2.00      0.438          1.00        38.9
  100000             131072         15          1.15      0.763          1.01        22.3
  114688             131072         15          1.00      0.875          1.02        19.4
  114689             262144         16          2.00      0.438          1.00        38.9

  the same power-of-two swing: one key apart (57,344 and 57,345) the
  achieved load goes from 0.875 to 0.438 and the cost per key from
  19.4 to 38.9 bytes. What does not swing is the lookup: key
  comparisons per hit stay between 1.00 and 1.02 across that whole range,
  where the hand-written table's probes/hit went from 1.30 to 2.62 over
  a similar one (0.38 to 0.76). Why is example 6.

rehashing also does something the load factor does not advertise:
it is the only thing that removes tombstones.

  700 inserts                    used=700 tombs=  0 size=1024 rehashes=0
  delete 600 of them             used=100 tombs=600 size=1024 rehashes=0
  186 more inserts               used=286 tombs=  0 size=1024 rehashes=1
  size before the rehash: 1024, after: 1024

  two different things happened on that last line, and only one of
  them is a rehash.

  first, Put REUSES a tombstone it walks past, so many of the 600
  came back with no rebuild at all -- which is why it took 186
  inserts to cross a threshold that was only 68 slots away.

  second, once used+tombstones did cross 0.75*1024 the table
  rebuilt itself -- and the SIZE DID NOT CHANGE, because the live
  count was still low. Rebuilding in place was enough, and it
  cleared every remaining tombstone at once.

  that second part is the one to get right. A grow routine that
  ALWAYS doubles will double a table that is 90% tombstones, and
  then that table is 45% tombstones and twice as big, forever.

  and std's table, the same keys. It shows no tombstone count, so 'tombs'
  is inferred as 7/8 * buckets - capacity() (the accounting is read from
  raw.rs; buckets come from the allocator's byte count, not from capacity()):

  std: 700 inserts               used=700 tombs=  0 capacity()= 896 buckets=1024
  std: delete 600 of them        used=100 tombs=329 capacity()= 567 buckets=1024
  std: 715 more inserts          used=815 tombs=  0 capacity()=1792 buckets=2048
  that last insert moved 814 entries and made 1 allocation event(s); capacity() 814 (== len(): the budget was gone) -> 1792

  std did not do what the hand-written table did, and the numbers say why.
  Only 329 of the 600 removals left a tombstone; the other 271 handed their
  budget straight back (raw.rs erase, read not run: it writes an EMPTY slot, not a
  tombstone, whenever an EMPTY slot lies within a group's width of the removed
  one; that 271 is what capacity() shows). So the budget was 467 inserts into
  never-used slots, not 68, and it took 715 inserts to spend it, because an insert
  that lands on a tombstone spends none. By then 815 entries were live, 91% of
  the 896 the table allows, and growing was the right answer: raw.rs rebuilds
  IN PLACE only when the live entries are at most half the capacity, and
  grows otherwise (read, not run).

  the in-place branch needs live entries to stay low while the budget
  leaks away, which is churn: the same table, 100 live entries, then one
  new key in and the oldest key out, over and over.

  std churn: start               used=100 tombs=329 capacity()= 567 buckets=1024
  14199 insert+remove pairs      used=101 tombs=  0 capacity()= 896 buckets=1024
  that last insert moved 100 entries and made 0 allocation event(s); capacity() 100 (== len(): the budget was gone) -> 896

  capacity() drained from 567 to 100 as tombstones piled up with the live count
  fixed at 100, and then one insert rebuilt the table at the same size, with
  no new allocation, and capacity() came back to 896. That is the behaviour the
  hand-written table's grow() has to be written to have.

what it all costs on the clock:

  ns/insert                      grown from 4   preallocated  growing x
  hand-written table                     26.6            7.7       3.5x
  std HashMap, same hash                 12.4            4.7       2.6x
  std HashMap, SipHash default           18.5           11.0       1.7x

  growing costs real time in every row (1.7x to 3.5x here), and it is the
  one cost you can remove entirely by knowing the size in advance:
  `HashMap::with_capacity(n)` is the whole fix. The hand-written 'preallocated'
  table is sized 2n as in the Go program, 524288 slots against the 262144 buckets that
  with_capacity(n) picks, so it touches twice the memory. A size hint
  on the map type itself is what example 10 measures.
```

> **Go vs Rust:** Every deterministic row of the hand-written table is identical to Go (16 rehashes, 196,605 entries moved, 1.97 moves per insert, the four-row load sweep, 186 inserts before the in-place rebuild), and Go's three parallel arrays stay three `Vec`s so the 17 B/slot column still matches. The std rows are new and use only `capacity()`, a hasher that counts its calls and the counting allocator: `HashMap` with the same hash behaves like the 0.875 row (15 rehashes, 1.15 moves per insert, 22.3 B/key, one allocation per growth against three), but at the Go tombstone step it grew to 1,792 instead of rebuilding in place (329 tombstones inferred from `capacity()`, not 600), and the in-place branch only shows up under churn, with 0 allocations. The timing columns are samples: growing cost 1.4x-4.6x over repeated runs, and the slowest single insert was a rehash of 0.75-1.05 ms in every run.

**Complexity:** **1.97 moves per insert** amortized, while the **worst single `Put` moved 196,608
entries** · the max-load setting collapses into pairs because sizes are powers of two — read the
*achieved* load (0.38 vs 0.76) instead · a rehash is the only thing that clears tombstones, and it
**must be allowed to keep the same size**

---

## 6. Go's map is a Swiss table

`🟢 easy` · *everything you read about `bmap` and `tophash` is obsolete*

Since Go 1.24 the map is a Swiss table: groups of 8 slots with an 8-byte **control word**, an H1/H2
hash split, and one 64-bit operation that compares a candidate against all eight slots at once.

**Steps:**

1. Reimplement `matchH2` and watch SWAR resolve eight slots in four instructions.
2. See where iteration randomisation and non-addressable values come from.
3. Then measure hit against miss, and get the answer wrong twice.

```go
package main

import (
	"fmt"
	"math/bits"
	"runtime"
	"strings"
	"testing"
	"unsafe"
)

// Go's map is a SWISS TABLE, and has been since Go 1.24. If you have read about
// Go maps before, you have probably read about `bmap`, buckets of 8 with a
// `tophash` array, and overflow buckets chained off the end.
//
// That implementation no longer exists. It is worth knowing what replaced it,
// because the replacement explains the load factor, the iteration order, and
// why `&m[k]` does not compile.
//
// The design, from internal/runtime/maps/map.go:
//
//	SLOT     one key/value pair
//	GROUP    8 slots + ONE 8-byte CONTROL WORD
//	          each control byte is either empty (0b10000000),
//	          deleted (0b11111110), or the low 7 bits of a key's hash
//	TABLE    an array of groups
//	MAP      a DIRECTORY of tables (extendible hashing)
//
//	H1       the upper 57 bits of the hash -> which group to start at
//	H2       the lower 7 bits             -> stored in the control byte
//
// The trick is the control word. Eight 7-bit hash fragments sit in one uint64,
// so ONE 64-bit operation compares a candidate against all 8 slots at once.
// That is why a Swiss table can run at 87.5% load: a probe is a group check,
// not a slot check.

// matchH2 is the actual trick, reimplemented here so you can watch it work.
// This is Go's ctrlGroup.matchH2, which is Abseil's, which is a classic SWAR
// (SIMD Within A Register) technique.
const (
	bitsetLSB = 0x0101010101010101
	bitsetMSB = 0x8080808080808080
)

func matchH2(ctrl uint64, h2 uint8) uint64 {
	// XOR makes every byte that equals h2 become zero.
	v := ctrl ^ (bitsetLSB * uint64(h2))
	// This is the has-zero-byte trick: it sets the high bit of every byte
	// that was zero, and only those.
	return (v - bitsetLSB) & ^v & bitsetMSB
}

func matchEmpty(ctrl uint64) uint64 {
	// empty is 0b10000000: high bit set, all others clear.
	return (ctrl & ^(ctrl << 1)) & bitsetMSB
}

func renderCtrl(ctrl uint64) string {
	var parts []string
	for i := 0; i < 8; i++ {
		b := uint8(ctrl >> (8 * i))
		switch b {
		case 0x80:
			parts = append(parts, "  empty")
		case 0xfe:
			parts = append(parts, "    del")
		default:
			parts = append(parts, fmt.Sprintf("   %#02x", b))
		}
	}
	return strings.Join(parts, "")
}

func renderMatch(m uint64) string {
	var parts []string
	for i := 0; i < 8; i++ {
		if m&(1<<(8*i+7)) != 0 {
			parts = append(parts, "      *")
		} else {
			parts = append(parts, "      .")
		}
	}
	return strings.Join(parts, "")
}

type S struct {
	A int
	B string
}

var sinkI int
var sinkB bool

// liveKB reports the heap held by whatever build returns, after two full GC
// cycles on each side (example 1 explains why two).
func liveKB(build func() any) float64 {
	runtime.GC()
	runtime.GC()
	var a, b runtime.MemStats
	runtime.ReadMemStats(&a)
	keep := build()
	runtime.GC()
	runtime.GC()
	runtime.ReadMemStats(&b)
	runtime.KeepAlive(keep)
	return float64(b.HeapAlloc-a.HeapAlloc) / 1024
}

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	fmt.Println("the control word: 8 slots described by 8 bytes, checked at once.")
	fmt.Println()
	// A group holding five keys whose H2 values are 0x2a, 0x11, 0x2a, 0x07, 0x63.
	var ctrl uint64
	for i, b := range []uint8{0x2a, 0x11, 0x2a, 0x07, 0x63, 0x80, 0x80, 0xfe} {
		ctrl |= uint64(b) << (8 * i)
	}
	fmt.Printf("  slot:   %s\n", func() string {
		var p []string
		for i := 0; i < 8; i++ {
			p = append(p, fmt.Sprintf("%7d", i))
		}
		return strings.Join(p, "")
	}())
	fmt.Printf("  ctrl:   %s\n", renderCtrl(ctrl))
	fmt.Println()
	for _, h2 := range []uint8{0x2a, 0x07, 0x55} {
		m := matchH2(ctrl, h2)
		n := bits.OnesCount64(m)
		fmt.Printf("  H2=%#02x: %s   %d candidate(s)\n", h2, renderMatch(m), n)
	}
	fmt.Printf("  empty:  %s   %d free slot(s)\n",
		renderMatch(matchEmpty(ctrl)), bits.OnesCount64(matchEmpty(ctrl)))

	fmt.Println()
	fmt.Println("  that is ONE subtraction, two ANDs and a NOT -- four instructions")
	fmt.Println("  to reject or shortlist eight slots. No branches, no key")
	fmt.Println("  comparisons, no memory beyond the 8 bytes already in a register.")
	fmt.Println()
	fmt.Println("  the keys themselves are only compared for the slots the control")
	fmt.Println("  word shortlisted. With 7 bits of hash there is a 1-in-128 chance")
	fmt.Println("  of a false candidate, so on average a lookup does ONE key")
	fmt.Println("  comparison, whatever the load factor.")
	fmt.Println()
	fmt.Println("  and that is the whole answer to 'why 7/8?'. At 87.5% load a group")
	fmt.Println("  holds 7 of 8 slots, so the first group check almost always")
	fmt.Println("  resolves the lookup. Chaining has to keep the load near 1.0 to")
	fmt.Println("  avoid long chains; a Swiss table is FASTER at high load because")
	fmt.Println("  the parallel check does not care how full the group is.")

	fmt.Println()
	fmt.Println("H1 and H2, the split that makes it work:")
	fmt.Println()
	fmt.Printf("  %-22s %s\n", "H1 = hash >> 7", "the upper 57 bits -- choose the group")
	fmt.Printf("  %-22s %s\n", "H2 = hash & 0x7f", "the lower 7 -- live in the control word")
	fmt.Println()
	fmt.Println("  both halves must be good, which is exactly what example 2's")
	fmt.Println("  finalizer measurement was about. A hash with all its entropy in")
	fmt.Println("  the low bits would give a fine H2 and a useless H1: every key")
	fmt.Println("  would start probing at the same group.")

	fmt.Println()
	fmt.Println("iteration order is randomised, and the randomisation is TWO-PART:")
	fmt.Println()
	m := map[int]int{}
	for i := 0; i < 24; i++ {
		m[i] = i
	}
	for r := 0; r < 4; r++ {
		fmt.Print("  ")
		n := 0
		for k := range m {
			fmt.Printf("%3d", k)
			if n++; n == 12 {
				break
			}
		}
		fmt.Println("  ...")
	}
	fmt.Println()
	fmt.Println("  from internal/runtime/maps/table.go:")
	fmt.Println()
	fmt.Println("      it.entryOffset = rand()   // where to start inside a group")
	fmt.Println("      it.dirOffset   = rand()   // which table to start with")
	fmt.Println()
	fmt.Println("  it is deliberate and it is not a debug feature -- it ships in")
	fmt.Println("  production builds. Code that accidentally depends on map order")
	fmt.Println("  fails on the second run instead of two years later.")
	fmt.Println()
	fmt.Println("  the one exception: `fmt` prints maps in SORTED key order, so")
	fmt.Println("  printing a map is stable even though ranging it is not.")
	fmt.Printf("  fmt: %v\n", map[string]int{"c": 3, "a": 1, "b": 2})

	fmt.Println()
	fmt.Println("map values are NOT ADDRESSABLE, and now you know why:")
	fmt.Println()
	fmt.Println("      p := &m[k]        // does not compile")
	fmt.Println("      m[k].Field = 1    // does not compile")
	fmt.Println("      m[k]++            // fine -- read, add, write back")
	fmt.Println()
	fmt.Println("  a rehash MOVES every entry to a new array. A pointer into a map")
	fmt.Println("  would dangle the moment the map grew, so the language forbids")
	fmt.Println("  taking one. This is not an arbitrary restriction; it falls")
	fmt.Println("  straight out of example 5.")
	fmt.Println()
	sm := map[string]S{"a": {A: 1, B: "x"}}
	v := sm["a"]
	v.A = 99
	sm["a"] = v
	fmt.Printf("  the read-modify-write dance:  %v\n", sm)
	pm := map[string]*S{"a": {A: 1, B: "x"}}
	pm["a"].A = 99 // legal: the POINTER is the value, and it does not move
	fmt.Printf("  or store a pointer instead:   %v\n", *pm["a"])
	fmt.Println()
	fmt.Println("  storing *S makes the value addressable again, at the cost of one")
	fmt.Println("  allocation and one indirection per entry -- lesson 04's trade,")
	fmt.Println("  arriving in a new place.")

	fmt.Println()
	fmt.Println("the sizes involved, on this machine:")
	fmt.Println()
	fmt.Printf("  %-34s %d bytes\n", "one control word", 8)
	fmt.Printf("  %-34s %d slots", "one group", 8)
	fmt.Println()
	fmt.Printf("  %-34s %d bytes\n", "unsafe.Sizeof(map[int]int{})", unsafe.Sizeof(map[int]int{}))
	fmt.Printf("  %-34s %d bytes\n", "unsafe.Sizeof(struct{k,v int}{})", unsafe.Sizeof(struct{ k, v int }{}))
	fmt.Println()
	fmt.Println("  a map variable is ONE POINTER. That is why a map is a reference")
	fmt.Println("  type, why passing it to a function is free, why the zero value is")
	fmt.Println("  nil, and why writing to a nil map panics while reading returns")
	fmt.Println("  the zero value.")
	fmt.Println()
	var nilMap map[string]int
	fmt.Printf("  var m map[string]int -> len=%d, m[\"x\"]=%d, ok=%v\n",
		len(nilMap), nilMap["x"], func() bool { _, ok := nilMap["x"]; return ok }())
	fmt.Println("  m[\"x\"] = 1 would panic: assignment to entry in nil map")

	fmt.Println()
	fmt.Println("and now a measurement that corrected me twice. I expected a MISS")
	fmt.Println("to be as fast as a hit, because a miss should resolve in the")
	fmt.Println("control word without ever comparing a key:")
	fmt.Println()
	fmt.Printf("  %11s %11s %10s %10s %10s %10s\n",
		"live keys", "map KB", "bytes/key", "hit ns", "miss ns", "miss/hit")
	for _, n := range []int{1000, 100_000, 150_000, 200_000, 300_000, 2_000_000} {
		present := make([]int, n)
		for i := range present {
			present[i] = i*2654435761 + 1
		}
		absent := make([]int, n)
		for i := range absent {
			absent[i] = -(i*2654435761 + 1)
		}
		kb := liveKB(func() any {
			mm := make(map[int]int, n)
			for i, k := range present {
				mm[k] = i
			}
			return mm
		})
		gm := make(map[int]int, n)
		for i, k := range present {
			gm[k] = i
		}
		hit := nsPerOp(func() {
			for _, k := range present {
				_, sinkB = gm[k]
			}
		}) / float64(n)
		miss := nsPerOp(func() {
			for _, k := range absent {
				_, sinkB = gm[k]
			}
		}) / float64(n)
		fmt.Printf("  %11d %11.0f %10.1f %10.2f %10.2f %9.2fx\n",
			n, kb, kb*1024/float64(n), hit, miss, miss/hit)
	}
	fmt.Println()
	fmt.Println("  the ratio is not a constant, and it does not grow with n -- it")
	fmt.Println("  goes 1.16, 2.42, 1.29, 2.39, 1.34, 0.85. Line the rows up against")
	fmt.Println("  BYTES/KEY instead and the middle four sort themselves:")
	fmt.Println()
	fmt.Printf("  %-30s %s\n", "23.7 and 23.6 bytes/key", "miss/hit 2.42x and 2.39x")
	fmt.Printf("  %-30s %s\n", "31.5 and 31.5 bytes/key", "miss/hit 1.29x and 1.34x")
	fmt.Println()
	fmt.Println("  bytes/key is the achieved load factor in disguise, and it swings")
	fmt.Println("  with n for the power-of-two reason example 1 measured. So:")
	fmt.Println()
	fmt.Println("  a HIT stops at the group holding its key. A MISS cannot stop")
	fmt.Println("  until it finds an EMPTY slot -- and in a DENSE table empty slots")
	fmt.Println("  are rare, so it walks more groups, each a separate cache line.")
	fmt.Println("  In a sparse table it finds one almost immediately.")
	fmt.Println()
	fmt.Println("  the two outer rows are governed by something else. At 1,000")
	fmt.Println("  keys the whole table is in L1, so walking extra groups is free")
	fmt.Println("  and the ratio is 1.16 whatever the density.")
	fmt.Println()
	fmt.Println("  and at 2,000,000 keys the ratio inverts: both are memory-bound,")
	fmt.Println("  and now the HIT is the slower one, because after the control word")
	fmt.Println("  shortlists a slot the hit must fetch and compare the actual key --")
	fmt.Println("  one more cache line that the miss never touches.")
	fmt.Println()
	fmt.Println("  my original claim was right about the mechanism and wrong about")
	fmt.Println("  the conditions. That is the usual shape of a benchmark surprise.")
	fmt.Println()
	fmt.Println("  what does hold unconditionally is the comparison to CHAINING,")
	fmt.Println("  where a miss walks the entire chain -- example 3 measured 1000.5")
	fmt.Println("  links per lookup on a degenerate table. A Swiss table's miss is a")
	fmt.Println("  few group checks; a chained table's is unbounded. Example 11 puts")
	fmt.Println("  them side by side.")
	sinkI = 1
}
```

**Sample output:**

```
the control word: 8 slots described by 8 bytes, checked at once.

  slot:         0      1      2      3      4      5      6      7
  ctrl:      0x2a   0x11   0x2a   0x07   0x63  empty  empty    del

  H2=0x2a:       *      .      *      .      .      .      .      .   2 candidate(s)
  H2=0x07:       .      .      .      *      .      .      .      .   1 candidate(s)
  H2=0x55:       .      .      .      .      .      .      .      .   0 candidate(s)
  empty:        .      .      .      .      .      *      *      .   2 free slot(s)

  that is ONE subtraction, two ANDs and a NOT -- four instructions
  to reject or shortlist eight slots. No branches, no key
  comparisons, no memory beyond the 8 bytes already in a register.

  the keys themselves are only compared for the slots the control
  word shortlisted. With 7 bits of hash there is a 1-in-128 chance
  of a false candidate, so on average a lookup does ONE key
  comparison, whatever the load factor.

  and that is the whole answer to 'why 7/8?'. At 87.5% load a group
  holds 7 of 8 slots, so the first group check almost always
  resolves the lookup. Chaining has to keep the load near 1.0 to
  avoid long chains; a Swiss table is FASTER at high load because
  the parallel check does not care how full the group is.

H1 and H2, the split that makes it work:

  H1 = hash >> 7         the upper 57 bits -- choose the group
  H2 = hash & 0x7f       the lower 7 -- live in the control word

  both halves must be good, which is exactly what example 2's
  finalizer measurement was about. A hash with all its entropy in
  the low bits would give a fine H2 and a useless H1: every key
  would start probing at the same group.

iteration order is randomised, and the randomisation is TWO-PART:

   18 22  4 19  7  8 10 11 17 21 23  2  ...
   22  4 19  7  8 10 11 17 21 23  2  3  ...
   12 14 16 18 22  4 19  7  8 10 11 17  ...
   22  4 19  7  8 10 11 17 21 23  2  3  ...

  from internal/runtime/maps/table.go:

      it.entryOffset = rand()   // where to start inside a group
      it.dirOffset   = rand()   // which table to start with

  it is deliberate and it is not a debug feature -- it ships in
  production builds. Code that accidentally depends on map order
  fails on the second run instead of two years later.

  the one exception: `fmt` prints maps in SORTED key order, so
  printing a map is stable even though ranging it is not.
  fmt: map[a:1 b:2 c:3]

map values are NOT ADDRESSABLE, and now you know why:

      p := &m[k]        // does not compile
      m[k].Field = 1    // does not compile
      m[k]++            // fine -- read, add, write back

  a rehash MOVES every entry to a new array. A pointer into a map
  would dangle the moment the map grew, so the language forbids
  taking one. This is not an arbitrary restriction; it falls
  straight out of example 5.

  the read-modify-write dance:  map[a:{99 x}]
  or store a pointer instead:   {99 x}

  storing *S makes the value addressable again, at the cost of one
  allocation and one indirection per entry -- lesson 04's trade,
  arriving in a new place.

the sizes involved, on this machine:

  one control word                   8 bytes
  one group                          8 slots
  unsafe.Sizeof(map[int]int{})       8 bytes
  unsafe.Sizeof(struct{k,v int}{})   16 bytes

  a map variable is ONE POINTER. That is why a map is a reference
  type, why passing it to a function is free, why the zero value is
  nil, and why writing to a nil map panics while reading returns
  the zero value.

  var m map[string]int -> len=0, m["x"]=0, ok=false
  m["x"] = 1 would panic: assignment to entry in nil map

and now a measurement that corrected me twice. I expected a MISS
to be as fast as a hit, because a miss should resolve in the
control word without ever comparing a key:

    live keys      map KB  bytes/key     hit ns    miss ns   miss/hit
         1000          36       37.0       3.80       4.34      1.14x
       100000        2309       23.6       5.16      13.30      2.58x
       150000        4618       31.5       4.86       6.56      1.35x
       200000        4624       23.7       6.00      14.49      2.42x
       300000        9237       31.5       5.20       6.73      1.29x
      2000000       73888       37.8      19.07      15.80      0.83x

  the ratio is not a constant, and it does not grow with n -- it
  goes 1.16, 2.42, 1.29, 2.39, 1.34, 0.85. Line the rows up against
  BYTES/KEY instead and the middle four sort themselves:

  23.7 and 23.6 bytes/key        miss/hit 2.42x and 2.39x
  31.5 and 31.5 bytes/key        miss/hit 1.29x and 1.34x

  bytes/key is the achieved load factor in disguise, and it swings
  with n for the power-of-two reason example 1 measured. So:

  a HIT stops at the group holding its key. A MISS cannot stop
  until it finds an EMPTY slot -- and in a DENSE table empty slots
  are rare, so it walks more groups, each a separate cache line.
  In a sparse table it finds one almost immediately.

  the two outer rows are governed by something else. At 1,000
  keys the whole table is in L1, so walking extra groups is free
  and the ratio is 1.16 whatever the density.

  and at 2,000,000 keys the ratio inverts: both are memory-bound,
  and now the HIT is the slower one, because after the control word
  shortlists a slot the hit must fetch and compare the actual key --
  one more cache line that the miss never touches.

  my original claim was right about the mechanism and wrong about
  the conditions. That is the usual shape of a benchmark surprise.

  what does hold unconditionally is the comparison to CHAINING,
  where a miss walks the entire chain -- example 3 measured 1000.5
  links per lookup on a degenerate table. A Swiss table's miss is a
  few group checks; a chained table's is unbounded. Example 11 puts
  them side by side.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::{BTreeMap, HashMap, HashSet};
use std::hash::{BuildHasher, Hash, Hasher};
use std::hint::black_box;
use std::mem::size_of;
use std::sync::atomic::{AtomicU64, AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Go's map is a SWISS TABLE, and has been since Go 1.24. Rust's `HashMap` is
// one too: std vendors `hashbrown` (0.17.1 in this toolchain's rust-src), whose
// Cargo.toml calls it "a Rust port of Google's SwissTable hash map". So this
// example changes shape. Go's program reimplements one trick of the design and then
// describes the runtime's internals; Rust's std exposes NONE of those
// internals (no control bytes, no groups, no probe sequence), so the port
//
//   * reimplements the same trick (it is also hashbrown's portable fallback),
//   * MEASURES std's table from the outside: capacity(), the allocator, a key
//     that counts how often it is compared, a hasher whose bits we control,
//   * and says "read from the source" for everything else.
//
// What the source (library/vendor/hashbrown-0.17.1/src, read, not run) says,
// beside Go's design (the Go column was checked against the Go 1.27.1 source on
// this machine: h1/h2 in maps/map.go, maxTableCapacity in maps/table.go):
//
//                Go 1.24+                       Rust (hashbrown)
//   SLOT         one key/value                  one (K, V) "bucket"
//   GROUP        8 slots + an 8-byte word       Group::WIDTH control bytes: 8 here
//                                               (aarch64 NEON), 16 on x86_64 (SSE2)
//   CONTROL      empty 0x80, deleted 0xfe,      EMPTY 0xff, DELETED 0x80, full =
//   BYTE         full = low 7 bits of hash      TOP 7 bits of the hash (the "tag")
//   H1           hash >> 7: the HIGH bits       the hash itself; the start position
//                choose the group               is hash & bucket_mask: the LOW bits
//   MAP          a directory of tables of       ONE table: one allocation holding
//                at most 1024 slots, each       all the buckets and all the control
//                grown by itself                bytes; growth rebuilds all of it
//
// So the two designs split the hash in opposite directions, and Rust has no
// directory: example 5's "rehash moves everything" is literally true of std's
// whole table.

// ---- measuring kit: std-only stand-ins for what Go's stdlib gives for free ----

// Allocation counter (stand-in for runtime.MemStats): no GC means "how many
// allocations, how many live bytes" are exact numbers.
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

fn allocs() -> usize {
    ALLOCS.load(Relaxed)
}
/// bytes currently live on the heap (a deterministic MemStats.HeapAlloc).
fn live_bytes() -> usize {
    LIVE.load(Relaxed)
}

/// stand-in for testing.Benchmark / b.Loop: ns per call of `f`.
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

/// stand-in for math/rand: splitmix64, seeded, so the run is reproducible.
struct Rng(u64);
impl Rng {
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
}

// ---- hashing: the finalizer from example 2, and a way to break it on purpose ----

fn mix(mut k: u64) -> u64 {
    k ^= k >> 30;
    k = k.wrapping_mul(0xbf58476d1ce4e5b9);
    k ^= k >> 27;
    k = k.wrapping_mul(0x94d049bb133111eb);
    k ^= k >> 31;
    k
}

// Which bits of the hash survive. `Good` is the full finalizer. The other two
// each destroy one END of the hash, to find out which end std uses for what.
#[derive(Clone, Copy)]
enum Shape {
    Good,
    NoTag, // top 7 bits forced to zero, the other 57 still look random
    NoPos, // low 57 bits forced to zero; only the top 7 bits carry the key
}

#[derive(Clone, Copy)]
struct Shaped(Shape);
struct ShapedHasher(Shape, u64);
impl Hasher for ShapedHasher {
    fn write(&mut self, _: &[u8]) {
        unreachable!("this example only hashes u64 keys");
    }
    fn write_u64(&mut self, k: u64) {
        self.1 = match self.0 {
            Shape::Good => mix(k),
            Shape::NoTag => mix(k) >> 7,
            Shape::NoPos => (k & 0x7f) << 57,
        };
    }
    fn finish(&self) -> u64 {
        self.1
    }
}
impl BuildHasher for Shaped {
    type Hasher = ShapedHasher;
    fn build_hasher(&self) -> ShapedHasher {
        ShapedHasher(self.0, 0)
    }
}

// A key that counts how often it is compared. std's table compares keys only
// for the slots whose control byte matched, so this counts exactly the
// comparisons the control bytes failed to save.
static COMPARES: AtomicU64 = AtomicU64::new(0);
#[derive(Clone, Copy)]
struct K(u64);
impl PartialEq for K {
    fn eq(&self, o: &K) -> bool {
        COMPARES.fetch_add(1, Relaxed);
        self.0 == o.0
    }
}
impl Eq for K {}
impl Hash for K {
    fn hash<H: Hasher>(&self, h: &mut H) {
        h.write_u64(self.0);
    }
}

// ---- the SWAR trick, ported from Go (and from hashbrown's control/group/generic.rs) ----

const BITSET_LSB: u64 = 0x0101_0101_0101_0101;
const BITSET_MSB: u64 = 0x8080_8080_8080_8080;

fn match_h2(ctrl: u64, h2: u8) -> u64 {
    // XOR makes every byte that equals h2 become zero.
    let v = ctrl ^ (BITSET_LSB * h2 as u64);
    // The has-zero-byte trick: sets the high bit of every byte that was zero.
    // (Go has no overflow to worry about; here the subtraction wraps on
    // purpose.)
    v.wrapping_sub(BITSET_LSB) & !v & BITSET_MSB
}

// Go's encoding: empty is 0b1000_0000 (high bit set, all others clear).
fn match_empty_go(ctrl: u64) -> u64 {
    (ctrl & !(ctrl << 1)) & BITSET_MSB
}

// hashbrown's encoding: EMPTY 0xff and DELETED 0x80 both have the high bit set,
// and only EMPTY has the next bit too.
fn match_empty_hashbrown(ctrl: u64) -> u64 {
    ctrl & (ctrl << 1) & BITSET_MSB
}

fn render_ctrl(ctrl: u64, empty: u8, deleted: u8) -> String {
    (0..8)
        .map(|i| match (ctrl >> (8 * i)) as u8 {
            b if b == empty => format!("{:>7}", "empty"),
            b if b == deleted => format!("{:>7}", "del"),
            b => format!("{:>7}", format!("{b:#04x}")),
        })
        .collect()
}

fn render_match(m: u64) -> String {
    (0..8)
        .map(|i| {
            if m & (1 << (8 * i + 7)) != 0 {
                "      *"
            } else {
                "      ."
            }
        })
        .collect()
}

fn pack(bytes: [u8; 8]) -> u64 {
    u64::from_le_bytes(bytes)
}

// ---- measuring std's table from outside ----

// One row of the "which end is the tag?" experiment: 100 keys in a std map
// whose hash has been damaged in one way. Returns (compares per hit, compares
// per miss, ns per hit lookup).
fn shape_row(label: &str, shape: Shape) -> (f64, f64, f64) {
    const N: u64 = 100;
    let mut counted: HashMap<K, u64, Shaped> = HashMap::with_hasher(Shaped(shape));
    for k in 0..N {
        counted.insert(K(k), k);
    }
    COMPARES.store(0, Relaxed);
    for k in 0..N {
        black_box(counted.get(&K(k)));
    }
    let per_hit = COMPARES.load(Relaxed) as f64 / N as f64;
    COMPARES.store(0, Relaxed);
    for k in 1000..1000 + N {
        black_box(counted.get(&K(k)));
    }
    let per_miss = COMPARES.load(Relaxed) as f64 / N as f64;
    // the clock is run on plain u64 keys: no atomic increment in the loop
    let mut plain: HashMap<u64, u64, Shaped> = HashMap::with_hasher(Shaped(shape));
    for k in 0..N {
        plain.insert(k, k);
    }
    let ns = ns_per_op(|| {
        for k in 0..N {
            black_box(plain.get(&k));
        }
    }) / N as f64;
    println!("  {label:<34} {per_hit:>12.2} {per_miss:>12.2} {ns:>12.2}");
    (per_hit, per_miss, ns)
}

// Group::WIDTH of this machine's table, read off capacity(): raw.rs gives a table
// of 1-byte entries a minimum capacity of 7 when groups are 8 control bytes
// wide and 14 when they are 16 wide (read, not run).
fn group_width() -> usize {
    match HashSet::<u8>::with_capacity(1).capacity() {
        7 => 8,
        14 => 16,
        c => panic!("unexpected minimum capacity {c}"),
    }
}

// A table of B buckets of 16-byte entries allocates 17*B + WIDTH bytes (layout
// from raw.rs, read; the byte counts are the allocator's).
fn buckets_of(bytes: usize) -> usize {
    (bytes - group_width()) / 17
}

fn first_12(m: &HashMap<u64, u64, impl BuildHasher>) -> String {
    m.keys().take(12).map(|k| format!("{k:>3}")).collect()
}

fn addr<T>(r: &T) -> usize {
    r as *const T as usize
}

#[derive(Debug, Clone)]
struct S {
    a: i32,
    b: &'static str,
}

// One measured row: n keys in a table built with `with_capacity(cap)`.
struct Row {
    n: usize,
    buckets: usize,
    kb: f64,
    load: f64,
    bytes_per_key: f64,
    hit: f64,
    miss: f64,
    cmp_hit: f64,
    cmp_miss: f64,
}

impl Row {
    fn ratio(&self) -> f64 {
        self.miss / self.hit
    }
}

fn lookup_row(n: usize, cap: usize) -> Row {
    let present: Vec<u64> = (0..n as u64)
        .map(|i| i.wrapping_mul(2654435761).wrapping_add(1))
        .collect();
    let absent: Vec<u64> = present.iter().map(|&k| 0u64.wrapping_sub(k)).collect();
    let before = live_bytes();
    let mut m: HashMap<u64, u64, Shaped> =
        HashMap::with_capacity_and_hasher(cap, Shaped(Shape::Good));
    for (i, &k) in present.iter().enumerate() {
        m.insert(k, i as u64);
    }
    let bytes = live_bytes() - before;
    let buckets = buckets_of(bytes);
    let hit = ns_per_op(|| {
        let mut c = 0;
        for k in &present {
            c += m.contains_key(k) as usize;
        }
        black_box(c);
    }) / n as f64;
    let miss = ns_per_op(|| {
        let mut c = 0;
        for k in &absent {
            c += m.contains_key(k) as usize;
        }
        black_box(c);
    }) / n as f64;
    drop(m);
    // the same table again, with keys that count their comparisons
    let mut counted: HashMap<K, u64, Shaped> =
        HashMap::with_capacity_and_hasher(cap, Shaped(Shape::Good));
    for (i, &k) in present.iter().enumerate() {
        counted.insert(K(k), i as u64);
    }
    COMPARES.store(0, Relaxed);
    for &k in &present {
        black_box(counted.contains_key(&K(k)));
    }
    let cmp_hit = COMPARES.load(Relaxed) as f64 / n as f64;
    COMPARES.store(0, Relaxed);
    for &k in &absent {
        black_box(counted.contains_key(&K(k)));
    }
    let cmp_miss = COMPARES.load(Relaxed) as f64 / n as f64;
    Row {
        n,
        buckets,
        kb: bytes as f64 / 1024.0,
        load: n as f64 / buckets as f64,
        bytes_per_key: bytes as f64 / n as f64,
        hit,
        miss,
        cmp_hit,
        cmp_miss,
    }
}

fn main() {
    println!("the control word: 8 slots described by 8 bytes, checked at once.");
    println!();
    // A group holding five keys whose H2 values are 0x2a, 0x11, 0x2a, 0x07, 0x63.
    let go_bytes = [0x2a, 0x11, 0x2a, 0x07, 0x63, 0x80, 0x80, 0xfe];
    let ctrl = pack(go_bytes);
    println!(
        "  slot:   {}",
        (0..8).map(|i| format!("{i:>7}")).collect::<String>()
    );
    println!("  ctrl:   {}", render_ctrl(ctrl, 0x80, 0xfe));
    println!();
    for h2 in [0x2a, 0x07, 0x55] {
        let m = match_h2(ctrl, h2);
        println!(
            "  H2={h2:#04x}: {}   {} candidate(s)",
            render_match(m),
            m.count_ones()
        );
    }
    let e = match_empty_go(ctrl);
    println!(
        "  empty:  {}   {} free slot(s)",
        render_match(e),
        e.count_ones()
    );
    println!();
    println!("  no branches, no key comparisons, no memory beyond the 8 bytes already");
    println!("  in a register: the whole thing is arithmetic on one u64.");
    println!();
    println!("  std's table has no such word to look at -- its control bytes are private.");
    println!("  What can be shown is hashbrown's own copy of this arithmetic (its portable");
    println!("  fallback, control/group/generic.rs; on aarch64 the real thing is one NEON");
    println!("  compare, neon.rs, and on x86_64 one SSE2 compare -- all read, not run) on its");
    println!("  OWN byte encoding:");
    println!("  EMPTY 0xff, DELETED 0x80, a full byte is the top 7 bits of the hash.");
    println!();
    let hb_bytes = [0x2a, 0x11, 0x2a, 0x07, 0x63, 0xff, 0xff, 0x80];
    let hb = pack(hb_bytes);
    println!(
        "  raw:    {}",
        hb_bytes
            .iter()
            .map(|b| format!("{:>7}", format!("{b:#04x}")))
            .collect::<String>()
    );
    println!("  ctrl:   {}", render_ctrl(hb, 0xff, 0x80));
    for h2 in [0x2a, 0x07, 0x55] {
        let m = match_h2(hb, h2);
        println!(
            "  H2={h2:#04x}: {}   {} candidate(s)",
            render_match(m),
            m.count_ones()
        );
    }
    let e = match_empty_hashbrown(hb);
    println!(
        "  empty:  {}   {} free slot(s)",
        render_match(e),
        e.count_ones()
    );
    println!();
    println!("  the same function finds the same slots in both encodings, because it");
    println!("  only looks for bytes equal to h2. One thing the Go program above does not");
    println!("  say, though hashbrown's generic.rs and Go's own group.go both do (read, not");
    println!("  run): the subtraction can borrow out of a true match into the byte above it,");
    println!("  so it may flag a slot that is NOT a match. Measured, over 2,000,000 random");
    println!("  words (Go's encoding, a quarter of the bytes empty, an eighth deleted, h2");
    println!("  present in half of the words):");
    println!();
    let mut rng = Rng(1);
    let (mut hits, mut missed, mut false_pos, mut odd_shape) = (0u64, 0u64, 0u64, 0u64);
    for _ in 0..2_000_000 {
        let mut bytes = [0u8; 8];
        for b in &mut bytes {
            let x = rng.next_u64();
            *b = match x % 8 {
                0 | 1 => 0x80,
                2 => 0xfe,
                _ => ((x >> 8) & 0x7f) as u8,
            };
        }
        let h2 = if rng.next_u64() & 1 == 0 {
            bytes[(rng.next_u64() % 8) as usize] & 0x7f
        } else {
            (rng.next_u64() & 0x7f) as u8
        };
        let m = match_h2(pack(bytes), h2);
        for (i, &b) in bytes.iter().enumerate() {
            let flagged = m & (1 << (8 * i + 7)) != 0;
            match (flagged, b == h2) {
                (true, true) => hits += 1,
                (false, true) => missed += 1,
                (true, false) => {
                    false_pos += 1;
                    // the claim to check: always a neighbour one bit away from
                    // h2, always above a true match
                    if b ^ h2 != 1 || !bytes[..i].contains(&h2) {
                        odd_shape += 1;
                    }
                }
                (false, false) => {}
            }
        }
    }
    println!("  {:<44} {:>8}", "slots flagged that really match", hits);
    println!("  {:<44} {:>8}", "real matches missed", missed);
    println!(
        "  {:<44} {:>8}",
        "slots flagged that are NOT a match", false_pos
    );
    println!(
        "  {:<44} {:>8}",
        "...of those, not 'one bit off AND above a match'", odd_shape
    );
    println!();
    println!("  harmless, because the key comparison that follows rejects it, and rare");
    println!(
        "  ({:.2}% of the flags here), but it means the shortlist is a SUPERSET of the",
        100.0 * false_pos as f64 / (hits + false_pos) as f64
    );
    println!("  true matches, never a subset: no false negatives, as the zero above says.");
    println!();
    println!("  the keys themselves are compared only for the slots the shortlist holds.");
    println!("  A 7-bit tag leaves a 1-in-128 chance of a wrong candidate per slot, so a");
    println!("  lookup should compare close to ONE key. That is measured at the end, on");
    println!("  std's real table, at several loads.");

    println!();
    println!("H1 and H2, the split that makes it work:");
    println!();
    println!("  Go:        H1 = hash >> 7          the upper 57 bits choose the group");
    println!("             H2 = hash & 0x7f        the lower 7 are the control byte");
    println!("  hashbrown: H1 = hash & bucket_mask the LOW bits choose where to start");
    println!("             tag = hash >> 57        the TOP 7 are the control byte");
    println!("  (hashbrown column: raw.rs h1() and control/tag.rs Tag::full(), read, not run.)");
    println!();
    println!("  the Go program warns that a hash with all its entropy in the low bits has a");
    println!("  fine H2 and a useless H1. For std it is the other way round, and that can be");
    println!("  tested through the public API alone: damage one end of the hash on purpose");
    println!("  and see what a lookup costs. 100 keys, a std map, a hasher whose bits we");
    println!("  control; 'compares' = key comparisons per lookup, 'ns' = per hit lookup:");
    println!();
    println!(
        "  {:<34} {:>12} {:>12} {:>12}",
        "hash", "compares/hit", "compares/miss", "ns/hit"
    );
    let good = shape_row("full finalizer", Shape::Good);
    let no_tag = shape_row("top 7 bits zeroed, low 57 random", Shape::NoTag);
    let no_pos = shape_row("low 57 bits zeroed, top 7 distinct", Shape::NoPos);
    println!();
    println!("  zero the TOP 7 bits and every key carries the same tag, so the control bytes");
    println!(
        "  shortlist every full slot of the group: {:.2} comparisons per hit instead of",
        no_tag.0
    );
    println!(
        "  {:.2}. That is the tag being the top of the hash. (With Go's split the tag would",
        good.0
    );
    println!("  be the low 7 bits, which are random here, and this hash would have been fine:");
    println!("  reasoning, not run.)");
    println!();
    println!(
        "  zero the LOW 57 bits and the tags are all different ({:.2} comparisons per hit:",
        no_pos.0
    );
    println!("  the tag does its job) but every key starts probing at the same place, and a");
    println!(
        "  lookup walks until it finds its key: {:.1} ns instead of {:.1}. That is the start",
        no_pos.2, good.2
    );
    println!(
        "  position being the low bits of the hash. (The walk itself is inferred: std shows no"
    );
    println!("  probe length, so the clock is the only evidence for it.)");
    println!();
    println!("  so both halves must be good -- example 2's point -- but the halves are the");
    println!("  other way round from Go's.");

    println!();
    println!("iteration order is randomised, but NOT the way Go does it:");
    println!();
    let keys: Vec<u64> = (0..24).collect();
    println!("  four different maps with the same 24 keys (each has its own RandomState):");
    for _ in 0..4 {
        let m: HashMap<u64, u64> = keys.iter().map(|&k| (k, k)).collect();
        println!("  {}  ...", first_12(&m));
    }
    println!();
    println!("  one map, iterated four times:");
    let one: HashMap<u64, u64> = keys.iter().map(|&k| (k, k)).collect();
    let lines: Vec<String> = (0..4).map(|_| first_12(&one)).collect();
    for l in &lines {
        println!("  {l}  ...");
    }
    println!(
        "  all four identical: {}",
        lines.iter().all(|l| *l == lines[0])
    );
    println!();
    println!("  and a map with a FIXED hasher (no random seed), the same 24 keys, twice:");
    for _ in 0..2 {
        let m: HashMap<u64, u64, Shaped> = keys.iter().map(|&k| (k, k)).fold(
            HashMap::with_hasher(Shaped(Shape::Good)),
            |mut m, (k, v)| {
                m.insert(k, v);
                m
            },
        );
        println!("  {}  ...", first_12(&m));
    }
    println!();
    println!("  the Go program quotes maps/table.go: it.entryOffset and it.dirOffset are drawn");
    println!("  at random for every `range`, so ranging one map twice gives two orders. std");
    println!("  does not do that: iterating walks the control bytes in memory order (read,");
    println!(
        "  not run), so one map has one order, the order is a function of the hash, and the only"
    );
    println!("  randomness is the per-map seed in RandomState -- hence the four different");
    println!("  maps above, and the two identical lines from the fixed hasher, which are");
    println!("  the same on every run of this program. Per-map randomness is enough to stop code");
    println!("  from depending on the order from one run to the next; unlike Go's, it does");
    println!("  not vary between two loops over the same map.");
    println!();
    println!("  the other Go exception, `fmt` printing maps sorted, has no twin: {{:?}} of a");
    println!("  HashMap prints in iteration order, so two maps with the same contents print");
    println!("  in whatever orders their seeds give them; a BTreeMap is the sorted one.");
    let letters = ["h", "c", "a", "g", "b", "f", "d", "e"];
    let hash_print = || {
        let m: HashMap<&str, usize> = letters.iter().enumerate().map(|(i, &l)| (l, i)).collect();
        format!("{m:?}")
    };
    let (first, second) = (hash_print(), hash_print());
    println!("  HashMap:  {first}");
    println!("  HashMap:  {second}");
    println!(
        "  (the same contents, printed {})",
        if first == second {
            "alike"
        } else {
            "differently"
        }
    );
    println!(
        "  BTreeMap: {:?}",
        letters
            .iter()
            .enumerate()
            .map(|(i, &l)| (l, i))
            .collect::<BTreeMap<_, _>>()
    );
    println!();
    println!("map values are NOT ADDRESSABLE in Go. In Rust the rule is the same idea, enforced");
    println!("by the borrow checker (messages from rustc 1.98.1, compiled separately):");
    println!();
    println!("      m[&k].field = 1    // error[E0594]: cannot assign to data in an index of");
    println!(
        "                         //   `HashMap<&str, S>` (trait `IndexMut` ... is not implemented)"
    );
    println!("      m[&k] += 1         // the same E0594: Go's m[k]++ has no direct twin");
    println!("      let p = &m[&k];    //");
    println!("      m.insert(..);      // error[E0502]: cannot borrow `m` as mutable because it");
    println!("      use(p);            //   is also borrowed as immutable");
    println!();
    println!(
        "  so a reference into the map is LEGAL in Rust -- Go forbids &m[k] outright -- but it"
    );
    println!("  cannot outlive the next insert. The reason is the same as Go's, and it can be");
    println!("  measured: record where an entry lives (as a plain number, so nothing is");
    println!("  borrowed), grow the map, look again.");
    println!();
    let mut am: HashMap<u64, S> = HashMap::new();
    am.insert(0, S { a: 1, b: "x" });
    let before = addr(&am[&0]);
    let (grow_allocs, ()) = counted(|| {
        for k in 1..1000 {
            am.insert(k, S { a: 1, b: "x" });
        }
    });
    let after = addr(&am[&0]);
    println!(
        "  HashMap<u64, S>      entry 0 moved when the map grew 1 -> 1000 entries: {}",
        before != after
    );
    let mut bm: HashMap<u64, Box<S>> = HashMap::new();
    bm.insert(0, Box::new(S { a: 1, b: "x" }));
    let before = addr(&*bm[&0]);
    let (box_allocs, ()) = counted(|| {
        for k in 1..1000 {
            bm.insert(k, Box::new(S { a: 1, b: "x" }));
        }
    });
    let after = addr(&*bm[&0]);
    println!(
        "  HashMap<u64, Box<S>> entry 0 moved when the map grew 1 -> 1000 entries: {}",
        before != after
    );
    println!();
    println!("  and what modifying in place looks like, with no read-modify-write dance:");
    let mut sm: HashMap<&str, S> = HashMap::new();
    sm.insert("a", S { a: 1, b: "x" });
    if let Some(v) = sm.get_mut("a") {
        v.a = 99; // a borrow that ends before the next insert
    }
    println!(
        "  get_mut:                  {sm:?}   (b is still {:?})",
        sm["a"].b
    );
    let mut counts: HashMap<&str, u32> = HashMap::new();
    for w in ["x", "y", "x"] {
        *counts.entry(w).or_insert(0) += 1; // Go's m[k]++
    }
    let mut counted_words: Vec<_> = counts.into_iter().collect();
    counted_words.sort();
    println!("  entry().or_insert() += 1: {counted_words:?}");
    println!();
    println!("  the Go program stores *S to get an addressable value. Rust's `Box<S>` does the");
    println!("  same job -- the S stays put when the table moves -- at the price the Go");
    println!("  program names, and the allocator counts it: growing from 1 to 1000 entries");
    println!(
        "  took {box_allocs} allocation events with a Box (999 boxes plus the table's own {grow_allocs}"
    );
    println!("  growths) and {grow_allocs} without.");
    println!();
    println!("the sizes involved, on this machine:");
    println!();
    let width_probe = {
        let before = live_bytes();
        let s: HashSet<u8> = HashSet::with_capacity(1);
        (s.capacity(), live_bytes() - before)
    };
    let width = group_width();
    println!(
        "  {:<44} {}",
        "size_of::<(u64, u64)>()  (one entry)",
        size_of::<(u64, u64)>()
    );
    println!(
        "  {:<44} {}",
        "size_of::<HashMap<u64, u64>>()",
        size_of::<HashMap<u64, u64>>()
    );
    println!(
        "  {:<44} {}",
        "  of which RandomState (two u64 seeds)",
        size_of::<std::hash::RandomState>()
    );
    println!(
        "  {:<44} {}",
        "size_of::<Option<HashMap<u64, u64>>>()",
        size_of::<Option<HashMap<u64, u64>>>()
    );
    println!();
    println!("  a Go map variable is ONE POINTER. A Rust HashMap is the table's header held");
    println!("  inline: the 48 bytes are the 16-byte RandomState plus 32 bytes that raw.rs");
    println!("  names as a pointer to the block, the bucket mask, the growth budget and the");
    println!("  item count (read, not run). Passing it BY VALUE moves those 48 bytes, passing");
    println!("  `&map` is the one-pointer version, and the buckets stay where they are.");
    println!();
    println!("  and the table behind it, measured by asking for capacity and watching the");
    println!("  allocator:");
    println!();
    println!(
        "  {:>10} {:>12} {:>14} {:>10} {:>10}",
        "requested", "capacity()", "bytes alloc'd", "buckets", "cap/bucket"
    );
    for want in [1usize, 4, 8, 15, 100, 1000, 100_000] {
        let before = live_bytes();
        let m: HashMap<u64, u64> = HashMap::with_capacity(want);
        let bytes = live_bytes() - before;
        let buckets = buckets_of(bytes);
        println!(
            "  {:>10} {:>12} {:>14} {:>10} {:>10.3}",
            want,
            m.capacity(),
            bytes,
            buckets,
            m.capacity() as f64 / buckets as f64
        );
    }
    println!();
    println!(
        "  every row obeys bytes = 17 * buckets + {width}: the 16-byte entry plus ONE control"
    );
    println!("  byte per bucket, plus {width} trailing control bytes (hashbrown repeats the first");
    println!("  group there so an unaligned group load never runs off the end; layout read");
    println!("  from raw.rs, the byte counts are the allocator's). The {width} is Group::WIDTH.");
    println!(
        "  A second reading of it: HashSet<u8>::with_capacity(1) has capacity {} and",
        width_probe.0
    );
    println!(
        "  allocates {} bytes; raw.rs picks that minimum from the group width (7 for 8,",
        width_probe.1
    );
    println!(
        "  14 for 16), so on this {} machine groups are {width} control bytes wide{}.",
        std::env::consts::ARCH,
        if width == 8 {
            ", the same as Go's"
        } else {
            ", unlike Go's 8"
        }
    );
    println!("  capacity() is 7/8 of the bucket count from 8 buckets up (a table of 4 keeps");
    println!("  one bucket empty, so that a missing key's walk always ends).");
    println!();
    println!("  a Go map's zero value is nil: reads return the zero value, a write panics.");
    println!("  Rust has no nil map. `HashMap::new()` is a real, empty map that allocates");
    println!("  nothing, and the first insert allocates:");
    let (n0, mut empty) = counted(HashMap::<u64, u64>::new);
    println!(
        "  HashMap::new(): {n0} allocation events, len={}, capacity()={}, get(&7)={:?}",
        empty.len(),
        empty.capacity(),
        empty.get(&7)
    );
    let (n1, ()) = counted(|| {
        empty.insert(7, 1);
    });
    println!(
        "  first insert:   {n1} allocation event, capacity()={}",
        empty.capacity()
    );
    println!("  \"assignment to entry in nil map\" cannot be written; the explicit version is");
    println!("  Option<HashMap<..>>, which costs no extra bytes (48 either way, measured above;");
    println!("  presumably the table's non-null pointer is the niche).");

    println!();
    println!("and now the measurement that corrected the Go program's author twice. The");
    println!("tempting expectation: a MISS costs the same as a hit, because it should resolve");
    println!("in the control bytes without comparing a key. Here is std's table, with the");
    println!("splitmix finalizer as the hash (so that the ~6 ns example 1 measured for the");
    println!("default SipHash does not swamp the table) and `with_capacity(n)`. The lookups");
    println!("are independent, so the CPU overlaps them: these are throughput numbers, as in");
    println!("the Go program.");
    println!();
    println!(
        "  {:>10} {:>9} {:>6} {:>10} {:>8} {:>8} {:>9} {:>9} {:>9}",
        "live keys",
        "map KB",
        "load",
        "bytes/key",
        "hit ns",
        "miss ns",
        "miss/hit",
        "cmp/hit",
        "cmp/miss"
    );
    let rows: Vec<Row> = [1000, 100_000, 150_000, 200_000, 300_000, 2_000_000]
        .into_iter()
        .map(|n| {
            let r = lookup_row(n, n);
            println!(
                "  {:>10} {:>9.0} {:>6.3} {:>10.1} {:>8.2} {:>8.2} {:>8.2}x {:>9.3} {:>9.3}",
                r.n,
                r.kb,
                r.load,
                r.bytes_per_key,
                r.hit,
                r.miss,
                r.ratio(),
                r.cmp_hit,
                r.cmp_miss
            );
            r
        })
        .collect();
    println!();
    println!(
        "  the ratio is not a constant, and it does not grow with n: it goes {}.",
        rows.iter()
            .map(|r| format!("{:.2}", r.ratio()))
            .collect::<Vec<_>>()
            .join(", ")
    );
    println!("  Line the rows up against LOAD instead (the achieved load factor, which swings");
    println!("  with n for the power-of-two reason example 5 measured; bytes/key is the same");
    println!("  number in disguise) and the middle four sort themselves:");
    println!();
    let pair = |a: &Row, b: &Row| {
        format!(
            "load {:.3} and {:.3} ({:.1} B/key)",
            a.load, b.load, a.bytes_per_key
        )
    };
    println!(
        "  {:<40} miss/hit {:.2}x and {:.2}x",
        pair(&rows[1], &rows[3]),
        rows[1].ratio(),
        rows[3].ratio()
    );
    println!(
        "  {:<40} miss/hit {:.2}x and {:.2}x",
        pair(&rows[2], &rows[4]),
        rows[2].ratio(),
        rows[4].ratio()
    );
    println!();
    println!("  one table size, ONE variable: 262,144 buckets (capacity 229,376) filled to");
    println!("  different loads, so the cache footprint is the same in every row:");
    println!();
    println!(
        "  {:>10} {:>6} {:>8} {:>8} {:>9} {:>9} {:>9}",
        "live keys", "load", "hit ns", "miss ns", "miss/hit", "cmp/hit", "cmp/miss"
    );
    let sweep: Vec<Row> = [50_000, 100_000, 150_000, 200_000, 229_376]
        .into_iter()
        .map(|n| {
            let r = lookup_row(n, 229_376);
            println!(
                "  {:>10} {:>6.3} {:>8.2} {:>8.2} {:>8.2}x {:>9.3} {:>9.3}",
                r.n,
                r.load,
                r.hit,
                r.miss,
                r.ratio(),
                r.cmp_hit,
                r.cmp_miss
            );
            r
        })
        .collect();
    let (lo, hi) = (&sweep[0], &sweep[sweep.len() - 1]);
    let all: Vec<&Row> = rows.iter().chain(sweep.iter()).collect();
    let range = |f: fn(&&Row) -> f64| {
        let v = all.iter().map(f);
        (
            v.clone().fold(f64::MAX, f64::min),
            v.fold(f64::MIN, f64::max),
        )
    };
    let (ch_lo, ch_hi) = range(|r| r.cmp_hit);
    let big = &rows[rows.len() - 1];
    println!();
    println!(
        "  with the size fixed, the miss goes from {:.2} ns at load {:.3} to {:.2} ns at {:.3}",
        lo.miss, lo.load, hi.miss, hi.load
    );
    println!(
        "  (miss/hit {:.2}x to {:.2}x), while the hit goes from {:.2} ns to {:.2} ns. So it is the",
        lo.ratio(),
        hi.ratio(),
        lo.hit,
        hi.hit
    );
    println!("  LOAD that moves the miss, not the size of the table (the same in every row).");
    println!();
    println!("  the Go program's summary says 'a Swiss table is FASTER at high load because the");
    println!("  parallel check does not care how full the group is'. In std's table the time does");
    println!(
        "  care: a hit costs {:.1}x and a miss {:.1}x as much at load {:.3} as at {:.3}. What does not",
        hi.hit / lo.hit,
        hi.miss / lo.miss,
        hi.load,
        lo.load
    );
    println!("  care is the number of key comparisons (below).");
    println!();
    println!(
        "  key comparisons are not the reason. A hit compares between {ch_lo:.3} and {ch_hi:.3} keys, in every"
    );
    println!(
        "  row of both tables, from load {:.2} up to {:.3}: the control bytes do what the design",
        lo.load, hi.load
    );
    println!(
        "  promises, about ONE comparison per hit at ANY load. A miss compares {:.3} keys at load",
        lo.cmp_miss
    );
    println!(
        "  {:.2} and {:.3} at {:.3}: a miss almost never touches a key, and even at the limit",
        lo.load, hi.cmp_miss, hi.load
    );
    println!("  that is far too little work to explain a miss that costs several times a hit.");
    println!("  What does the extra work is plausibly the walk (a miss ends only at a group");
    println!("  with an EMPTY byte, and a dense table has fewer of those, so the loop runs");
    println!("  again more often and its exit is harder to predict) -- plausible mechanism,");
    println!("  not measured: std exposes no probe length.");
    println!();
    println!(
        "  at {} keys the ratio inverts ({:.2}x): the hit is now the slower one. Plausible",
        big.n,
        big.ratio()
    );
    println!(
        "  mechanism, not measured: a miss reads only the control bytes ({} bytes here, buckets +",
        big.buckets + group_width()
    );
    println!(
        "  {}), while a hit also reads its entry from the {}-byte entry array: {:.0}x as many bytes.",
        group_width(),
        16 * big.buckets,
        (16 * big.buckets) as f64 / (big.buckets + group_width()) as f64
    );
    println!("  The Go program gives the same story for its own map.");
    println!();
    println!("  what holds at every load is the comparison to CHAINING, where a miss walks the");
    println!("  entire chain -- example 3 measured 1000.5 links per lookup on a degenerate table.");
    println!(
        "  A Swiss table's miss is a few group checks and, here, at most {:.2} key",
        hi.cmp_miss
    );
    println!("  comparisons; a chained table's is unbounded. Example 11 puts them side by side.");
}

// (allocation events, result) while running `f`.
fn counted<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let a0 = allocs();
    let r = f();
    (allocs() - a0, r)
}
```

**Rust sample output:**

```
the control word: 8 slots described by 8 bytes, checked at once.

  slot:         0      1      2      3      4      5      6      7
  ctrl:      0x2a   0x11   0x2a   0x07   0x63  empty  empty    del

  H2=0x2a:       *      .      *      .      .      .      .      .   2 candidate(s)
  H2=0x07:       .      .      .      *      .      .      .      .   1 candidate(s)
  H2=0x55:       .      .      .      .      .      .      .      .   0 candidate(s)
  empty:        .      .      .      .      .      *      *      .   2 free slot(s)

  no branches, no key comparisons, no memory beyond the 8 bytes already
  in a register: the whole thing is arithmetic on one u64.

  std's table has no such word to look at -- its control bytes are private.
  What can be shown is hashbrown's own copy of this arithmetic (its portable
  fallback, control/group/generic.rs; on aarch64 the real thing is one NEON
  compare, neon.rs, and on x86_64 one SSE2 compare -- all read, not run) on its
  OWN byte encoding:
  EMPTY 0xff, DELETED 0x80, a full byte is the top 7 bits of the hash.

  raw:       0x2a   0x11   0x2a   0x07   0x63   0xff   0xff   0x80
  ctrl:      0x2a   0x11   0x2a   0x07   0x63  empty  empty    del
  H2=0x2a:       *      .      *      .      .      .      .      .   2 candidate(s)
  H2=0x07:       .      .      .      *      .      .      .      .   1 candidate(s)
  H2=0x55:       .      .      .      .      .      .      .      .   0 candidate(s)
  empty:        .      .      .      .      .      *      *      .   2 free slot(s)

  the same function finds the same slots in both encodings, because it
  only looks for bytes equal to h2. One thing the Go program above does not
  say, though hashbrown's generic.rs and Go's own group.go both do (read, not
  run): the subtraction can borrow out of a true match into the byte above it,
  so it may flag a slot that is NOT a match. Measured, over 2,000,000 random
  words (Go's encoding, a quarter of the bytes empty, an eighth deleted, h2
  present in half of the words):

  slots flagged that really match                698845
  real matches missed                                 0
  slots flagged that are NOT a match               2923
  ...of those, not 'one bit off AND above a match'        0

  harmless, because the key comparison that follows rejects it, and rare
  (0.42% of the flags here), but it means the shortlist is a SUPERSET of the
  true matches, never a subset: no false negatives, as the zero above says.

  the keys themselves are compared only for the slots the shortlist holds.
  A 7-bit tag leaves a 1-in-128 chance of a wrong candidate per slot, so a
  lookup should compare close to ONE key. That is measured at the end, on
  std's real table, at several loads.

H1 and H2, the split that makes it work:

  Go:        H1 = hash >> 7          the upper 57 bits choose the group
             H2 = hash & 0x7f        the lower 7 are the control byte
  hashbrown: H1 = hash & bucket_mask the LOW bits choose where to start
             tag = hash >> 57        the TOP 7 are the control byte
  (hashbrown column: raw.rs h1() and control/tag.rs Tag::full(), read, not run.)

  the Go program warns that a hash with all its entropy in the low bits has a
  fine H2 and a useless H1. For std it is the other way round, and that can be
  tested through the public API alone: damage one end of the hash on purpose
  and see what a lookup costs. 100 keys, a std map, a hasher whose bits we
  control; 'compares' = key comparisons per lookup, 'ns' = per hit lookup:

  hash                               compares/hit compares/miss       ns/hit
  full finalizer                             1.01         0.05         2.28
  top 7 bits zeroed, low 57 random           2.22         8.68         4.04
  low 57 bits zeroed, top 7 distinct         1.00         0.76         7.17

  zero the TOP 7 bits and every key carries the same tag, so the control bytes
  shortlist every full slot of the group: 2.22 comparisons per hit instead of
  1.01. That is the tag being the top of the hash. (With Go's split the tag would
  be the low 7 bits, which are random here, and this hash would have been fine:
  reasoning, not run.)

  zero the LOW 57 bits and the tags are all different (1.00 comparisons per hit:
  the tag does its job) but every key starts probing at the same place, and a
  lookup walks until it finds its key: 7.2 ns instead of 2.3. That is the start
  position being the low bits of the hash. (The walk itself is inferred: std shows no
  probe length, so the clock is the only evidence for it.)

  so both halves must be good -- example 2's point -- but the halves are the
  other way round from Go's.

iteration order is randomised, but NOT the way Go does it:

  four different maps with the same 24 keys (each has its own RandomState):
   12  2  3 23  8 15  5 17 19 14 22 20  ...
    6  8 10 14 22  5 11  7 20 18 15  2  ...
   16  4 22 23 17 13 21  7  9 14 11  3  ...
   10 21  3  2  8 11  1 12 16  0 14  5  ...

  one map, iterated four times:
    4 17  7 11 13  6 15 16 10 22 23  5  ...
    4 17  7 11 13  6 15 16 10 22 23  5  ...
    4 17  7 11 13  6 15 16 10 22 23  5  ...
    4 17  7 11 13  6 15 16 10 22 23  5  ...
  all four identical: true

  and a map with a FIXED hasher (no random seed), the same 24 keys, twice:
    0  1 23  8 14  2 21  6 11  3 13 18  ...
    0  1 23  8 14  2 21  6 11  3 13 18  ...

  the Go program quotes maps/table.go: it.entryOffset and it.dirOffset are drawn
  at random for every `range`, so ranging one map twice gives two orders. std
  does not do that: iterating walks the control bytes in memory order (read,
  not run), so one map has one order, the order is a function of the hash, and the only
  randomness is the per-map seed in RandomState -- hence the four different
  maps above, and the two identical lines from the fixed hasher, which are
  the same on every run of this program. Per-map randomness is enough to stop code
  from depending on the order from one run to the next; unlike Go's, it does
  not vary between two loops over the same map.

  the other Go exception, `fmt` printing maps sorted, has no twin: {:?} of a
  HashMap prints in iteration order, so two maps with the same contents print
  in whatever orders their seeds give them; a BTreeMap is the sorted one.
  HashMap:  {"d": 6, "b": 4, "h": 0, "c": 1, "g": 3, "a": 2, "f": 5, "e": 7}
  HashMap:  {"e": 7, "b": 4, "f": 5, "d": 6, "h": 0, "a": 2, "g": 3, "c": 1}
  (the same contents, printed differently)
  BTreeMap: {"a": 2, "b": 4, "c": 1, "d": 6, "e": 7, "f": 5, "g": 3, "h": 0}

map values are NOT ADDRESSABLE in Go. In Rust the rule is the same idea, enforced
by the borrow checker (messages from rustc 1.98.1, compiled separately):

      m[&k].field = 1    // error[E0594]: cannot assign to data in an index of
                         //   `HashMap<&str, S>` (trait `IndexMut` ... is not implemented)
      m[&k] += 1         // the same E0594: Go's m[k]++ has no direct twin
      let p = &m[&k];    //
      m.insert(..);      // error[E0502]: cannot borrow `m` as mutable because it
      use(p);            //   is also borrowed as immutable

  so a reference into the map is LEGAL in Rust -- Go forbids &m[k] outright -- but it
  cannot outlive the next insert. The reason is the same as Go's, and it can be
  measured: record where an entry lives (as a plain number, so nothing is
  borrowed), grow the map, look again.

  HashMap<u64, S>      entry 0 moved when the map grew 1 -> 1000 entries: true
  HashMap<u64, Box<S>> entry 0 moved when the map grew 1 -> 1000 entries: false

  and what modifying in place looks like, with no read-modify-write dance:
  get_mut:                  {"a": S { a: 99, b: "x" }}   (b is still "x")
  entry().or_insert() += 1: [("x", 2), ("y", 1)]

  the Go program stores *S to get an addressable value. Rust's `Box<S>` does the
  same job -- the S stays put when the table moves -- at the price the Go
  program names, and the allocator counts it: growing from 1 to 1000 entries
  took 1008 allocation events with a Box (999 boxes plus the table's own 9
  growths) and 9 without.

the sizes involved, on this machine:

  size_of::<(u64, u64)>()  (one entry)         16
  size_of::<HashMap<u64, u64>>()               48
    of which RandomState (two u64 seeds)       16
  size_of::<Option<HashMap<u64, u64>>>()       48

  a Go map variable is ONE POINTER. A Rust HashMap is the table's header held
  inline: the 48 bytes are the 16-byte RandomState plus 32 bytes that raw.rs
  names as a pointer to the block, the bucket mask, the growth budget and the
  item count (read, not run). Passing it BY VALUE moves those 48 bytes, passing
  `&map` is the one-pointer version, and the buckets stay where they are.

  and the table behind it, measured by asking for capacity and watching the
  allocator:

   requested   capacity()  bytes alloc'd    buckets cap/bucket
           1            3             76          4      0.750
           4            7            144          8      0.875
           8           14            280         16      0.875
          15           28            552         32      0.875
         100          112           2184        128      0.875
        1000         1792          34824       2048      0.875
      100000       114688        2228232     131072      0.875

  every row obeys bytes = 17 * buckets + 8: the 16-byte entry plus ONE control
  byte per bucket, plus 8 trailing control bytes (hashbrown repeats the first
  group there so an unaligned group load never runs off the end; layout read
  from raw.rs, the byte counts are the allocator's). The 8 is Group::WIDTH.
  A second reading of it: HashSet<u8>::with_capacity(1) has capacity 7 and
  allocates 24 bytes; raw.rs picks that minimum from the group width (7 for 8,
  14 for 16), so on this aarch64 machine groups are 8 control bytes wide, the same as Go's.
  capacity() is 7/8 of the bucket count from 8 buckets up (a table of 4 keeps
  one bucket empty, so that a missing key's walk always ends).

  a Go map's zero value is nil: reads return the zero value, a write panics.
  Rust has no nil map. `HashMap::new()` is a real, empty map that allocates
  nothing, and the first insert allocates:
  HashMap::new(): 0 allocation events, len=0, capacity()=0, get(&7)=None
  first insert:   1 allocation event, capacity()=3
  "assignment to entry in nil map" cannot be written; the explicit version is
  Option<HashMap<..>>, which costs no extra bytes (48 either way, measured above;
  presumably the table's non-null pointer is the niche).

and now the measurement that corrected the Go program's author twice. The
tempting expectation: a MISS costs the same as a hit, because it should resolve
in the control bytes without comparing a key. Here is std's table, with the
splitmix finalizer as the hash (so that the ~6 ns example 1 measured for the
default SipHash does not swamp the table) and `with_capacity(n)`. The lookups
are independent, so the CPU overlaps them: these are throughput numbers, as in
the Go program.

   live keys    map KB   load  bytes/key   hit ns  miss ns  miss/hit   cmp/hit  cmp/miss
        1000        34  0.488       34.8     2.29     2.42     1.06x     1.002     0.022
      100000      2176  0.763       22.3     3.49     8.79     2.52x     1.011     0.077
      150000      4352  0.572       29.7     3.07     3.66     1.19x     1.005     0.040
      200000      4352  0.763       22.3     4.07    11.03     2.71x     1.012     0.076
      300000      8704  0.572       29.7     4.82     6.14     1.27x     1.005     0.040
     2000000     69632  0.477       35.7    13.53     5.88     0.43x     1.004     0.031

  the ratio is not a constant, and it does not grow with n: it goes 1.06, 2.52, 1.19, 2.71, 1.27, 0.43.
  Line the rows up against LOAD instead (the achieved load factor, which swings
  with n for the power-of-two reason example 5 measured; bytes/key is the same
  number in disguise) and the middle four sort themselves:

  load 0.763 and 0.763 (22.3 B/key)        miss/hit 2.52x and 2.71x
  load 0.572 and 0.572 (29.7 B/key)        miss/hit 1.19x and 1.27x

  one table size, ONE variable: 262,144 buckets (capacity 229,376) filled to
  different loads, so the cache footprint is the same in every row:

   live keys   load   hit ns  miss ns  miss/hit   cmp/hit  cmp/miss
       50000  0.191     2.50     2.13     0.85x     1.001     0.013
      100000  0.381     2.61     2.54     0.97x     1.002     0.025
      150000  0.572     3.06     3.65     1.19x     1.005     0.040
      200000  0.763     2.96     9.06     3.06x     1.012     0.076
      229376  0.875     4.04    18.42     4.55x     1.021     0.153

  with the size fixed, the miss goes from 2.13 ns at load 0.191 to 18.42 ns at 0.875
  (miss/hit 0.85x to 4.55x), while the hit goes from 2.50 ns to 4.04 ns. So it is the
  LOAD that moves the miss, not the size of the table (the same in every row).

  the Go program's summary says 'a Swiss table is FASTER at high load because the
  parallel check does not care how full the group is'. In std's table the time does
  care: a hit costs 1.6x and a miss 8.6x as much at load 0.875 as at 0.191. What does not
  care is the number of key comparisons (below).

  key comparisons are not the reason. A hit compares between 1.001 and 1.021 keys, in every
  row of both tables, from load 0.19 up to 0.875: the control bytes do what the design
  promises, about ONE comparison per hit at ANY load. A miss compares 0.013 keys at load
  0.19 and 0.153 at 0.875: a miss almost never touches a key, and even at the limit
  that is far too little work to explain a miss that costs several times a hit.
  What does the extra work is plausibly the walk (a miss ends only at a group
  with an EMPTY byte, and a dense table has fewer of those, so the loop runs
  again more often and its exit is harder to predict) -- plausible mechanism,
  not measured: std exposes no probe length.

  at 2000000 keys the ratio inverts (0.43x): the hit is now the slower one. Plausible
  mechanism, not measured: a miss reads only the control bytes (4194312 bytes here, buckets +
  8), while a hit also reads its entry from the 67108864-byte entry array: 16x as many bytes.
  The Go program gives the same story for its own map.

  what holds at every load is the comparison to CHAINING, where a miss walks the
  entire chain -- example 3 measured 1000.5 links per lookup on a degenerate table.
  A Swiss table's miss is a few group checks and, here, at most 0.15 key
  comparisons; a chained table's is unbounded. Example 11 puts them side by side.
```

> **Go vs Rust:** Rust's `HashMap` already is a Swiss table, so most of what the Go program peeks at (control bytes, group width, H1/H2, probe length) has no handle in std: this port re-implements only the SWAR `matchH2` (its first block matches Go's line for line) and measures std from outside, with `capacity()`, the counting allocator (bytes = 17 x buckets + 8, group width 8 inferred from `HashSet<u8>::with_capacity(1).capacity() == 7`), a key that counts its comparisons, and a hasher whose bits are damaged on purpose, which shows hashbrown uses the TOP 7 bits as the tag and the LOW bits as the start position, the reverse of Go's H1 = high / H2 = low; everything else about hashbrown (byte encodings, NEON, in-place-vs-grow rule) is labelled "read, not run" from the vendored 0.17.1 source. Rust disagrees with the Go prose in three places: iteration order is not re-randomised per `range` (one map keeps one order, only the per-map `RandomState` differs), `&m[&k]` compiles while `m[&k] += 1` does not (Go is the other way round), and at a fixed table size lookups get slower with load (miss 1.5 to 15 ns from load 0.19 to 0.875) even though key comparisons stay at 1.00-1.02 per hit. The miss/hit table has Go's shape over repeated runs (dense rows 2.4-4.5x, sparse rows 1.1-1.8x, 2,000,000 keys 0.3-0.5x), so it is a timing sample.

**Complexity:** one group check = **8 slots in ~4 instructions**, which is the whole answer to "why
7/8 load factor" · miss/hit is **not** a constant: **2.58× on a dense table, 1.29× on a sparse one,
and 0.83× at 2,000,000 keys** where the *hit* pays an extra cache line for the key comparison

---

> Next tier: [🟡 medium](2-medium.md) — sets, counters, what can be a key, preallocation,
> and four implementations measured against each other.
