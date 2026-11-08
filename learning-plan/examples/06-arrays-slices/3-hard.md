# Step 06 — Dynamic Arrays & Slices · 🔴 Hard

Examples **12–16**: choosing a structure, moving the cost somewhere else, and the capstone that
applies all three passes.

> ⚠️ **Example 16 is a `go test` package** — four files, run with `go test`. Examples 12–15 are
> `go run` programs. Sample output is from an Apple M4, Go 1.26.3.

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

## 12. Sorted slice vs map, with the workload as a variable

`🔴 hard` · *The honest comparison*

The textbook answer — "sorted slice is Θ(n) to insert, a map is Θ(1), use the map" — ignores every
constant lesson 04 measured. So measure it, with the ratio of lookups to insertions as a knob.

**Steps:**

1. Compare lookup-only on an already-built collection.
2. Then compare *building* it, at 0, 1, 10 and 100 lookups per insertion.
3. Read the ratio column, not the winner column.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// "Maintain a collection I can search" is the most common structure decision
// there is, and the textbook answer -- sorted slice is Theta(n) to insert, a map
// is Theta(1), use the map -- ignores the constants that lesson 04 measured.
//
// Here is the honest comparison, with the WORKLOAD as a variable. That turns
// out to be the thing that decides it.

var sinkB bool

type sortedSlice struct{ xs []int }

func (s *sortedSlice) Insert(v int) {
	i, found := slices.BinarySearch(s.xs, v)
	if found {
		return
	}
	s.xs = append(s.xs, 0)
	copy(s.xs[i+1:], s.xs[i:])
	s.xs[i] = v
}

func (s *sortedSlice) Contains(v int) bool {
	_, ok := slices.BinarySearch(s.xs, v)
	return ok
}

type hashSet struct{ m map[int]struct{} }

func newHashSet() *hashSet { return &hashSet{m: make(map[int]struct{})} }

func (h *hashSet) Insert(v int)        { h.m[v] = struct{}{} }
func (h *hashSet) Contains(v int) bool { _, ok := h.m[v]; return ok }

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// buildAndQuery runs a workload with the given ratio of lookups to insertions
// and reports the total nanoseconds for the whole batch.
func buildAndQuery(n, lookupsPerInsert int, insert func(int), contains func(int) bool) float64 {
	rng := rand.New(rand.NewPCG(1, 2))
	vals := rng.Perm(n)
	probes := make([]int, 256)
	for i := range probes {
		probes[i] = rng.IntN(n)
	}

	return nsPerOp(func() {
		p := 0
		for _, v := range vals {
			insert(v)
			for k := 0; k < lookupsPerInsert; k++ {
				sinkB = contains(probes[p&255])
				p++
			}
		}
	})
}

func main() {
	fmt.Println("lookup only, on an already-built collection of n elements:")
	fmt.Println()
	fmt.Printf("  %10s %16s %14s %10s\n", "n", "sorted slice ns", "map ns", "winner")

	rng := rand.New(rand.NewPCG(3, 4))
	for _, n := range []int{8, 64, 512, 4096, 65536} {
		ss := &sortedSlice{xs: make([]int, n)}
		hs := newHashSet()
		for i := 0; i < n; i++ {
			ss.xs[i] = i * 2
			hs.Insert(i * 2)
		}

		probes := make([]int, 256)
		for i := range probes {
			probes[i] = rng.IntN(n) * 2
		}
		p := 0
		pick := func() int { p = (p + 1) & 255; return probes[p] }

		tS := nsPerOp(func() { sinkB = ss.Contains(pick()) })
		tM := nsPerOp(func() { sinkB = hs.Contains(pick()) })

		w := "map"
		if tS < tM {
			w = "slice"
		}
		fmt.Printf("  %10d %16.1f %14.1f %10s\n", n, tS, tM, w)
	}

	fmt.Println()
	fmt.Println("now the part the textbook comparison leaves out -- BUILDING it,")
	fmt.Println("with a varying number of lookups per insertion:")
	fmt.Println()
	fmt.Printf("  %8s %10s %16s %14s %12s\n", "n", "lookups", "sorted slice ns", "map ns", "slice/map")

	for _, n := range []int{1000, 8000} {
		for _, lk := range []int{0, 1, 10, 100} {
			ss := &sortedSlice{}
			tS := buildAndQuery(n, lk, ss.Insert, ss.Contains)

			hs := newHashSet()
			tM := buildAndQuery(n, lk, hs.Insert, hs.Contains)

			fmt.Printf("  %8d %10d %16.0f %14.0f %11.1fx\n", n, lk, tS, tM, tS/tM)
		}
	}

	fmt.Println()
	fmt.Println("read that honestly: the map wins EVERY row. It is faster to look up")
	fmt.Println("(4.9 ns flat vs 15.5 ns and climbing) and enormously faster to build,")
	fmt.Println("because every sorted insert is a Theta(n) memmove and building by")
	fmt.Println("insertion is therefore Theta(n^2) (example 5).")
	fmt.Println()
	fmt.Println("what the ratio column shows is the gap NARROWING as lookups come to")
	fmt.Println("dominate -- roughly 7x down to 3x at n=8000 -- because the expensive")
	fmt.Println("inserts get amortised over many cheap, cache-friendly searches.")
	fmt.Println("It narrows. It does not cross over. Do not talk yourself into a")
	fmt.Println("sorted slice on the grounds of lookup speed; lesson 04, example 15")
	fmt.Println("reached exactly the same conclusion from the other direction.")
	fmt.Println()
	fmt.Println("but notice what is missing from every row: the case where you have")
	fmt.Println("all the data up front. Then it is not Theta(n^2) at all --")
	fmt.Println("append everything and slices.Sort once, Theta(n log n), and the")
	fmt.Println("sorted slice wins on memory (4.5x, lesson 04) AND supports range")
	fmt.Println("queries the map cannot answer at any price.")

	fmt.Println()
	fmt.Println("the decision, then, is not 'which is faster'. It is:")
	fmt.Println()
	fmt.Println("  do inserts and lookups interleave?   -> map")
	fmt.Println("  can you build once, then query?      -> sorted slice")
	fmt.Println("  need order, range, or rank?          -> sorted slice, no contest")
	fmt.Println("  need BOTH interleaved AND ordered?   -> a balanced tree (lesson 19)")
	fmt.Println("                                          or a skip list (lesson 37)")
}
```

**Sample output:**

```
lookup only, on an already-built collection of n elements:

           n  sorted slice ns         map ns     winner
           8              5.0            4.1        map
          64              7.5            4.5        map
         512             10.1            4.7        map
        4096             12.5            4.9        map
       65536             15.6            4.8        map

now the part the textbook comparison leaves out -- BUILDING it,
with a varying number of lookups per insertion:

         n    lookups  sorted slice ns         map ns    slice/map
      1000          0            12172           5063         2.4x
      1000          1            22342           9084         2.5x
      1000         10           121835          45975         2.6x
      1000        100          1019437         410740         2.5x
      8000          0           309780          46367         6.7x
      8000          1           626193          82478         7.6x
      8000         10          1500474         388955         3.9x
      8000        100         10415886        3471335         3.0x

read that honestly: the map wins EVERY row. It is faster to look up
(4.9 ns flat vs 15.5 ns and climbing) and enormously faster to build,
because every sorted insert is a Theta(n) memmove and building by
insertion is therefore Theta(n^2) (example 5).

what the ratio column shows is the gap NARROWING as lookups come to
dominate -- roughly 7x down to 3x at n=8000 -- because the expensive
inserts get amortised over many cheap, cache-friendly searches.
It narrows. It does not cross over. Do not talk yourself into a
sorted slice on the grounds of lookup speed; lesson 04, example 15
reached exactly the same conclusion from the other direction.

but notice what is missing from every row: the case where you have
all the data up front. Then it is not Theta(n^2) at all --
append everything and slices.Sort once, Theta(n log n), and the
sorted slice wins on memory (4.5x, lesson 04) AND supports range
queries the map cannot answer at any price.

the decision, then, is not 'which is faster'. It is:

  do inserts and lookups interleave?   -> map
  can you build once, then query?      -> sorted slice
  need order, range, or rank?          -> sorted slice, no contest
  need BOTH interleaved AND ordered?   -> a balanced tree (lesson 19)
                                          or a skip list (lesson 37)
```

The map wins **every row**. What the ratio column shows is the gap *narrowing* as lookups dominate —
roughly 7× down to 3× — because the expensive inserts get amortised over many cheap searches. It
narrows; it does not cross over. [Lesson 04, example 15](../04-go-memory-model/3-hard.md#15-sorted-slice-vs-map--the-honest-comparison)
reached the same conclusion from the other direction.

So don't choose a sorted slice for lookup speed. Choose it for **memory** (4.5×), **ordering**, and
**range queries** — and if you have the data up front, sort once and it isn't Θ(n²) at all.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashSet;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// "Maintain a collection I can search" is the most common structure decision
// there is, and the textbook answer -- sorted slice is Theta(n) to insert, a
// hash set is Theta(1), use the hash set -- ignores the constants that lesson 04
// measured.
//
// Here is the honest comparison, with the WORKLOAD as a variable. That turns
// out to be the thing that decides it.
//
// Go passes `insert`/`contains` closures around; Rust expresses the same thing
// as a trait with two methods, and the generic functions below are compiled
// once per concrete collection (monomorphised) -- no indirect call in the loop.

trait Collection {
    fn insert(&mut self, v: i64);
    fn contains(&self, v: i64) -> bool;
}

struct SortedSlice {
    xs: Vec<i64>,
}

impl Collection for SortedSlice {
    fn insert(&mut self, v: i64) {
        // binary_search returns Err(i) with the insertion point when v is absent
        if let Err(i) = self.xs.binary_search(&v) {
            self.xs.insert(i, v); // push + shift the tail right: Theta(n)
        }
    }
    fn contains(&self, v: i64) -> bool {
        self.xs.binary_search(&v).is_ok()
    }
}

// HashSet<i64> is a HashMap<i64, ()> underneath -- Go's map[int]struct{}.
struct HashSetOfInts {
    m: HashSet<i64>,
}

impl Collection for HashSetOfInts {
    fn insert(&mut self, v: i64) {
        self.m.insert(v);
    }
    fn contains(&self, v: i64) -> bool {
        self.m.contains(&v)
    }
}

// ---- stand-ins for what Go's stdlib gives for free ----

// testing.Benchmark / b.Loop: ns per call of `f`
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

// math/rand: splitmix64, seeded, so runs are reproducible (not Go's PCG: the
// permutation differs from Go's, the algorithm and the counts do not)
struct Rng(u64);
impl Rng {
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    /// like rand.IntN(n)
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
    /// like rand.Perm: Fisher-Yates
    fn shuffle<T>(&mut self, xs: &mut [T]) {
        for i in (1..xs.len()).rev() {
            xs.swap(i, self.below(i as u64 + 1) as usize);
        }
    }
}

// runtime.MemStats: a counting allocator that tracks live heap bytes. The trait
// is `unsafe` to implement; every method just forwards to the system allocator.
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

/// Heap bytes held by whatever `make` returns (the value is handed back so it
/// stays alive while we look at the number).
fn held_bytes<T>(make: impl FnOnce() -> T) -> (usize, T) {
    let before = LIVE.load(Relaxed);
    let v = make();
    (LIVE.load(Relaxed) - before, v)
}

// ---- the workload ----

/// The values to insert (a random permutation of 0..n) and 256 probe values.
fn workload(n: usize) -> (Vec<i64>, Vec<i64>) {
    let mut rng = Rng(1);
    let mut vals: Vec<i64> = (0..n as i64).collect();
    rng.shuffle(&mut vals);
    let probes = (0..256).map(|_| rng.below(n as u64) as i64).collect();
    (vals, probes)
}

/// build_and_query runs a workload with the given ratio of lookups to
/// insertions and reports the total nanoseconds for the whole batch.
///
/// Every timed iteration starts from a FRESH, empty collection (`fresh()`), so
/// every insert really inserts. If the collection were built once outside the
/// timed closure, iteration 2 onward would "insert" values that are already
/// there and be measuring lookups, not building.
fn build_and_query<C: Collection>(
    n: usize,
    lookups_per_insert: usize,
    fresh: impl Fn() -> C,
) -> f64 {
    let (vals, probes) = workload(n);
    ns_per_op(|| {
        let mut c = fresh();
        let mut p = 0;
        for &v in &vals {
            c.insert(v);
            for _ in 0..lookups_per_insert {
                black_box(c.contains(probes[p & 255]));
                p += 1;
            }
        }
        black_box(&c); // building and dropping the collection is part of the cost
    })
}

fn main() {
    println!("lookup only, on an already-built collection of n elements:");
    println!();
    println!(
        "  {:>10} {:>16} {:>14} {:>10}",
        "n", "sorted slice ns", "hash set ns", "winner"
    );

    let mut rng = Rng(3);
    let mut lookup_rows: Vec<(usize, f64, f64)> = Vec::new();
    for n in [8usize, 64, 512, 4096, 65536] {
        let ss = SortedSlice {
            xs: (0..n as i64).map(|i| i * 2).collect(),
        };
        let hs = HashSetOfInts {
            m: (0..n as i64).map(|i| i * 2).collect(),
        };

        let probes: Vec<i64> = (0..256).map(|_| rng.below(n as u64) as i64 * 2).collect();
        let mut p = 0;
        let mut pick = || {
            p = (p + 1) & 255;
            probes[p]
        };

        let t_s = ns_per_op(|| {
            black_box(ss.contains(pick()));
        });
        let t_m = ns_per_op(|| {
            black_box(hs.contains(pick()));
        });

        let w = if t_s < t_m { "slice" } else { "hash set" };
        println!("  {n:>10} {t_s:>16.1} {t_m:>14.1} {w:>10}");
        lookup_rows.push((n, t_s, t_m));
    }

    println!();
    println!("now the part the textbook comparison leaves out -- BUILDING it,");
    println!("with a varying number of lookups per insertion:");
    println!();
    println!(
        "  {:>8} {:>10} {:>16} {:>14} {:>12}",
        "n", "lookups", "sorted slice ns", "hash set ns", "slice/set"
    );

    // (n, lookups per insert, slice ns, set ns)
    let mut build_rows: Vec<(usize, usize, f64, f64)> = Vec::new();
    for n in [1000usize, 8000] {
        for lk in [0usize, 1, 10, 100] {
            let t_s = build_and_query(n, lk, || SortedSlice { xs: Vec::new() });
            let t_m = build_and_query(n, lk, || HashSetOfInts { m: HashSet::new() });
            println!(
                "  {n:>8} {lk:>10} {t_s:>16.0} {t_m:>14.0} {:>11.1}x",
                t_s / t_m
            );
            build_rows.push((n, lk, t_s, t_m));
        }
    }

    // Everything printed below is computed from the rows above, so it cannot
    // disagree with them.
    let list = |ns: Vec<usize>| {
        let parts: Vec<String> = ns.iter().map(|n| n.to_string()).collect();
        parts.join(", ")
    };
    let slice_wins: Vec<usize> = lookup_rows
        .iter()
        .filter(|r| r.1 < r.2)
        .map(|r| r.0)
        .collect();
    let set_wins: Vec<usize> = lookup_rows
        .iter()
        .filter(|r| r.1 >= r.2)
        .map(|r| r.0)
        .collect();
    let set_lo = lookup_rows.iter().map(|r| r.2).fold(f64::MAX, f64::min);
    let set_hi = lookup_rows.iter().map(|r| r.2).fold(0.0, f64::max);
    let (first, last) = (lookup_rows[0], lookup_rows[lookup_rows.len() - 1]);

    println!();
    println!("read that honestly -- lookups first:");
    println!();
    println!(
        "  the sorted slice wins the lookup at n = {}",
        list(slice_wins.clone())
    );
    println!(
        "  the hash set wins the lookup at n = {}",
        list(set_wins.clone())
    );
    println!(
        "  the set's cost is nearly flat ({set_lo:.1} to {set_hi:.1} ns: hash the key, probe a group);"
    );
    println!(
        "  the slice's climbs: {:.1} ns at n={}, {:.1} ns at n={} (log2(n) steps over a bigger array).",
        first.1, first.0, last.1, last.0
    );
    match (slice_wins.last(), set_wins.first()) {
        (Some(a), Some(b)) => {
            println!("  So the crossover sits between n={a} and n={b}: 'hash set wins lookups'");
            println!("  needs n big enough for the binary search to outgrow one hash.");
        }
        (None, _) => println!("  So the hash set wins every lookup row here."),
        (_, None) => println!("  So the sorted slice wins every lookup row here."),
    }

    println!();
    println!("and building, where every sorted insert is a Theta(n) memmove:");
    println!();
    for n in [1000usize, 8000] {
        let rows: Vec<&(usize, usize, f64, f64)> = build_rows.iter().filter(|r| r.0 == n).collect();
        let ratio = |r: &&(usize, usize, f64, f64)| r.2 / r.3;
        match rows.iter().find(|r| ratio(r) < 1.0) {
            Some(r) => println!(
                "  n={n}: the slice overtakes the hash set at {} lookups per insert ({:.1}x)",
                r.1,
                ratio(r)
            ),
            None => println!(
                "  n={n}: the hash set wins every row; slice/set narrows from {:.1}x to {:.1}x",
                ratio(&rows[0]),
                ratio(&rows[rows.len() - 1])
            ),
        }
    }
    println!();
    println!("  Why the crossover moves right as n grows: inserting a random permutation,");
    println!("  the i-th insert shifts about i/2 elements, so the average shift over the");
    println!("  whole build is n/4 elements. Insert cost grows with n; a lookup grows only");
    println!("  with log n; the hash set's insert stays O(1) amortised. Lookups per insert");
    println!("  are what amortise the shift, which is why the ratio narrows (first row to");
    println!("  last) as they grow.");

    let at = |n: usize| build_rows.iter().find(|r| r.0 == n && r.1 == 0).unwrap();
    let (small, big) = (at(1000), at(8000));
    println!();
    println!("  n 1000 -> 8000 is 8x the elements; with no lookups the build took");
    println!(
        "    {:.0}x longer for the sorted slice, {:.0}x for the hash set.",
        big.2 / small.2,
        big.3 / small.3
    );
    println!("    8x is linear growth, 64x a pure Theta(n^2). The hash set grew about linearly;");
    println!("    the slice grew far beyond that -- the quadratic term showing.");

    // ---- the case the tables leave out: all the data up front ----
    println!();
    println!("but notice what is missing from every row: the case where you have all the");
    println!("data up front. Then it is not Theta(n^2) at all -- copy everything in and");
    println!("sort once, Theta(n log n):");
    println!();

    let (vals, _) = workload(8000);
    let t_sort = ns_per_op(|| {
        let mut xs = vals.clone();
        xs.sort_unstable();
        black_box(&xs);
    });
    let (t_ins, t_set) = (at(8000).2, at(8000).3);
    println!("  n=8000, no lookups, building the same set three ways:");
    println!("    sorted slice, one insert at a time   {t_ins:>10.0} ns");
    println!("    hash set, one insert at a time       {t_set:>10.0} ns");
    println!("    clone + sort_unstable once           {t_sort:>10.0} ns");

    let n = 65536;
    let (slice_bytes, _slice) = held_bytes(|| (0..n as i64).collect::<Vec<i64>>());
    let (set_bytes, _set) = held_bytes(|| (0..n as i64).collect::<HashSet<i64>>());
    println!();
    println!(
        "  memory for {n} i64 keys: slice {slice_bytes} bytes ({} per key),",
        slice_bytes / n
    );
    println!(
        "  hash set {set_bytes} bytes ({:.1} per key): {:.2}x the slice.",
        set_bytes as f64 / n as f64,
        set_bytes as f64 / slice_bytes as f64
    );

    let mut sorted = vals;
    sorted.sort_unstable();
    let lo = sorted.partition_point(|&x| x < 1000);
    let hi = sorted.partition_point(|&x| x < 2000);
    println!();
    println!(
        "  and a range query: values in [1000, 2000) -> {} found with two binary searches",
        hi - lo
    );
    println!("  (partition_point). A HashSet has no order, so it would scan every element.");

    println!();
    println!("the decision, then, is not 'which is faster'. It is:");
    println!();
    println!("  do inserts and lookups interleave?   -> hash set, unless n is small and lookups");
    println!(
        "                                          dominate: then measure (see the n=1000 rows)"
    );
    println!("  can you build once, then query?      -> sorted slice");
    println!("  need order, range, or rank?          -> sorted slice, no contest");
    println!(
        "  need BOTH interleaved AND ordered?   -> a balanced tree (lesson 19; std has BTreeSet)"
    );
    println!("                                          or a skip list (lesson 37)");
}
```

**Rust sample output:**

```
lookup only, on an already-built collection of n elements:

           n  sorted slice ns    hash set ns     winner
           8              1.1            4.8      slice
          64              2.1            5.0      slice
         512              3.5            5.0      slice
        4096              5.2            5.1   hash set
       65536             16.1            4.9   hash set

now the part the textbook comparison leaves out -- BUILDING it,
with a varying number of lookups per insertion:

         n    lookups  sorted slice ns    hash set ns    slice/set
      1000          0            39205          19764         2.0x
      1000          1            43266          30159         1.4x
      1000         10            76237         133460         0.6x
      1000        100           402801         656721         0.6x
      8000          0          1432853         159777         9.0x
      8000          1          1480514         244932         6.0x
      8000         10          1871178         991146         1.9x
      8000        100          6572380        4409131         1.5x

read that honestly -- lookups first:

  the sorted slice wins the lookup at n = 8, 64, 512
  the hash set wins the lookup at n = 4096, 65536
  the set's cost is nearly flat (4.8 to 5.1 ns: hash the key, probe a group);
  the slice's climbs: 1.1 ns at n=8, 16.1 ns at n=65536 (log2(n) steps over a bigger array).
  So the crossover sits between n=512 and n=4096: 'hash set wins lookups'
  needs n big enough for the binary search to outgrow one hash.

and building, where every sorted insert is a Theta(n) memmove:

  n=1000: the slice overtakes the hash set at 10 lookups per insert (0.6x)
  n=8000: the hash set wins every row; slice/set narrows from 9.0x to 1.5x

  Why the crossover moves right as n grows: inserting a random permutation,
  the i-th insert shifts about i/2 elements, so the average shift over the
  whole build is n/4 elements. Insert cost grows with n; a lookup grows only
  with log n; the hash set's insert stays O(1) amortised. Lookups per insert
  are what amortise the shift, which is why the ratio narrows (first row to
  last) as they grow.

  n 1000 -> 8000 is 8x the elements; with no lookups the build took
    37x longer for the sorted slice, 8x for the hash set.
    8x is linear growth, 64x a pure Theta(n^2). The hash set grew about linearly;
    the slice grew far beyond that -- the quadratic term showing.

but notice what is missing from every row: the case where you have all the
data up front. Then it is not Theta(n^2) at all -- copy everything in and
sort once, Theta(n log n):

  n=8000, no lookups, building the same set three ways:
    sorted slice, one insert at a time      1432853 ns
    hash set, one insert at a time           159777 ns
    clone + sort_unstable once                45526 ns

  memory for 65536 i64 keys: slice 524288 bytes (8 per key),
  hash set 1179656 bytes (18.0 per key): 2.25x the slice.

  and a range query: values in [1000, 2000) -> 1000 found with two binary searches
  (partition_point). A HashSet has no order, so it would scan every element.

the decision, then, is not 'which is faster'. It is:

  do inserts and lookups interleave?   -> hash set, unless n is small and lookups
                                          dominate: then measure (see the n=1000 rows)
  can you build once, then query?      -> sorted slice
  need order, range, or rank?          -> sorted slice, no contest
  need BOTH interleaved AND ordered?   -> a balanced tree (lesson 19; std has BTreeSet)
                                          or a skip list (lesson 37)
```

> **Go vs Rust:** The conclusion changed, and not because of the language. The Go program builds each collection once, outside the benchmark loop, so from the second iteration on every `Insert` finds its key and returns (I counted 1,000 real inserts among 86,427,000 calls at n=1000), which means Go's "build" rows time lookups on a full collection; this twin builds a fresh collection per iteration, so it times real building. Measured that way the sorted slice wins the lookup at small n (about 1 ns at n=8 here; Go's sample says 5) and overtakes the set at n=1000 from 10 lookups per insert, so "the map wins every row" and "it does not cross over" do not hold for this twin. The hash set is 2.25x the slice's memory (`HashSet<i64>`: 18 bytes per key against 8), not the 4.5x of Go's `map`.

**Complexity:** sorted slice — find Θ(log n), insert Θ(n) · map — Θ(1) average, ~4.5× the memory, no order

---

## 13. The gap buffer

`🔴 hard` · *Moving the cost where nobody looks*

A dynamic array is Θ(n) to insert in the middle. A text editor inserts in the middle on every
keystroke. The gap buffer's answer: keep the array, but leave the **free space at the cursor**.

**Steps:**

1. Implement the buffer with `gapStart`/`gapEnd`, and render the gap so you can watch it move.
2. `Insert` fills one slot of the gap; `MoveTo` slides the gap.
3. Benchmark front-insertion against a plain `[]byte`.

```go
package main

import (
	"fmt"
	"strings"
	"testing"
)

// A dynamic array is Theta(n) to insert in the middle. A text editor inserts in
// the middle on EVERY KEYSTROKE. So how does one hold a 10 MB file and stay
// responsive?
//
// The GAP BUFFER: keep the array, but leave the free space in the MIDDLE,
// exactly where the cursor is. Then a keystroke is Theta(1) -- it just fills one
// slot of the gap. Moving the cursor is what costs, and it costs only the
// distance moved.
//
//   [ h e l l o _ _ _ _ _ _  w o r l d ]
//                ^^^^^^^^^^^
//               gapStart  gapEnd
//
// This is a real structure -- Emacs is built on one -- and it is the cleanest
// example of the idea that you can move a cost somewhere the workload does not
// look.

type GapBuffer struct {
	buf      []byte
	gapStart int // first free slot
	gapEnd   int // one past the last free slot
}

func NewGapBuffer(gap int) *GapBuffer {
	return &GapBuffer{buf: make([]byte, gap), gapStart: 0, gapEnd: gap}
}

func (g *GapBuffer) Len() int { return len(g.buf) - (g.gapEnd - g.gapStart) }

// grow doubles the buffer, placing a fresh gap at the cursor.
func (g *GapBuffer) grow() {
	newGap := len(g.buf)
	if newGap < 8 {
		newGap = 8
	}
	bigger := make([]byte, len(g.buf)+newGap)
	copy(bigger, g.buf[:g.gapStart])
	copy(bigger[g.gapStart+newGap:], g.buf[g.gapEnd:])
	g.buf = bigger
	g.gapEnd = g.gapStart + newGap
}

// Insert writes one byte at the cursor. Theta(1) amortized -- no shifting at all.
func (g *GapBuffer) Insert(c byte) {
	if g.gapStart == g.gapEnd {
		g.grow()
	}
	g.buf[g.gapStart] = c
	g.gapStart++
}

// Delete removes the byte before the cursor. Theta(1): just widen the gap.
func (g *GapBuffer) Delete() {
	if g.gapStart > 0 {
		g.gapStart--
	}
}

// MoveTo puts the cursor at position pos, moving the gap there. Theta(distance).
func (g *GapBuffer) MoveTo(pos int) {
	for g.gapStart > pos { // move left
		g.gapStart--
		g.gapEnd--
		g.buf[g.gapEnd] = g.buf[g.gapStart]
	}
	for g.gapStart < pos { // move right
		g.buf[g.gapStart] = g.buf[g.gapEnd]
		g.gapStart++
		g.gapEnd++
	}
}

func (g *GapBuffer) String() string {
	return string(g.buf[:g.gapStart]) + string(g.buf[g.gapEnd:])
}

// debug renders the gap visibly.
func (g *GapBuffer) debug() string {
	var b strings.Builder
	b.WriteString(string(g.buf[:g.gapStart]))
	b.WriteString("[")
	b.WriteString(strings.Repeat("_", g.gapEnd-g.gapStart))
	b.WriteString("]")
	b.WriteString(string(g.buf[g.gapEnd:]))
	return b.String()
}

// --- the naive alternative: a plain []byte ---

type plainBuffer struct{ buf []byte }

func (p *plainBuffer) InsertAt(pos int, c byte) {
	p.buf = append(p.buf, 0)
	copy(p.buf[pos+1:], p.buf[pos:])
	p.buf[pos] = c
}

var sinkS string

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

func main() {
	fmt.Println("watch the gap move as you type:")
	fmt.Println()
	g := NewGapBuffer(4)
	for _, c := range "hello" {
		g.Insert(byte(c))
	}
	fmt.Printf("  type 'hello'        %-22s -> %q\n", g.debug(), g.String())

	g.MoveTo(0)
	fmt.Printf("  cursor to start     %-22s\n", g.debug())

	for _, c := range ">> " {
		g.Insert(byte(c))
	}
	fmt.Printf("  type '>> '          %-22s -> %q\n", g.debug(), g.String())

	g.MoveTo(g.Len())
	for _, c := range "!" {
		g.Insert(byte(c))
	}
	fmt.Printf("  cursor to end, '!'  %-22s -> %q\n", g.debug(), g.String())

	g.Delete()
	fmt.Printf("  backspace           %-22s -> %q\n", g.debug(), g.String())

	fmt.Println()
	fmt.Println("now the workload that matters: typing n characters at the FRONT of a")
	fmt.Println("document -- the worst case for a plain array, the best for a gap buffer.")
	fmt.Println()
	fmt.Printf("  %10s %18s %18s %12s\n", "n", "plain []byte ns", "gap buffer ns", "speedup")

	for _, n := range []int{1000, 4000, 16000} {
		tPlain := nsPerOp(func() {
			p := &plainBuffer{}
			for i := 0; i < n; i++ {
				p.InsertAt(0, 'x') // always at the front: shift everything
			}
			sinkS = string(p.buf[:1])
		})
		tGap := nsPerOp(func() {
			gb := NewGapBuffer(8)
			for i := 0; i < n; i++ {
				gb.MoveTo(0) // the cursor is already there after the first move
				gb.Insert('x')
			}
			sinkS = gb.String()[:1]
		})
		fmt.Printf("  %10d %18.0f %18.0f %11.1fx\n", n, tPlain, tGap, tPlain/tGap)
	}

	fmt.Println()
	fmt.Println("the plain array is Theta(n^2): every insert shifts everything after it.")
	fmt.Println("The gap buffer is Theta(n): the gap is already at the cursor, so each")
	fmt.Println("keystroke fills one slot.")

	fmt.Println()
	fmt.Println("what it costs, honestly:")
	fmt.Println()
	fmt.Println("  MoveTo is Theta(distance). Jumping from the top of a file to the")
	fmt.Println("  bottom copies the whole document. A gap buffer is a bet that edits")
	fmt.Println("  CLUSTER -- which, for a human typing, they overwhelmingly do.")
	fmt.Println()
	fmt.Println("  it is also still one contiguous array, so it keeps every locality")
	fmt.Println("  property from lesson 04. That is why editors preferred it to a")
	fmt.Println("  linked list of lines for decades.")
	fmt.Println()
	fmt.Println("the transferable idea: you cannot remove the Theta(n) shift from an")
	fmt.Println("array, but you CAN choose where it happens -- and put it somewhere")
	fmt.Println("the workload rarely looks.")
}
```

**Sample output:**

```
watch the gap move as you type:

  type 'hello'        hello[_______]         -> "hello"
  cursor to start     [_______]hello        
  type '>> '          >> [____]hello         -> ">> hello"
  cursor to end, '!'  >> hello![___]         -> ">> hello!"
  backspace           >> hello[____]         -> ">> hello"

now the workload that matters: typing n characters at the FRONT of a
document -- the worst case for a plain array, the best for a gap buffer.

           n    plain []byte ns      gap buffer ns      speedup
        1000               8473               1662         5.1x
        4000              82542               6018        13.7x
       16000            1172861              23254        50.4x

the plain array is Theta(n^2): every insert shifts everything after it.
The gap buffer is Theta(n): the gap is already at the cursor, so each
keystroke fills one slot.

what it costs, honestly:

  MoveTo is Theta(distance). Jumping from the top of a file to the
  bottom copies the whole document. A gap buffer is a bet that edits
  CLUSTER -- which, for a human typing, they overwhelmingly do.

  it is also still one contiguous array, so it keeps every locality
  property from lesson 04. That is why editors preferred it to a
  linked list of lines for decades.

the transferable idea: you cannot remove the Theta(n) shift from an
array, but you CAN choose where it happens -- and put it somewhere
the workload rarely looks.
```

**5.4× → 12.1× → 46.0×** as n grows — the signature of Θ(n²) against Θ(n).

What it costs, honestly: `MoveTo` is Θ(distance), so jumping from the top of a file to the bottom
copies the whole document. A gap buffer is a bet that edits **cluster** — which, for a human typing,
they overwhelmingly do. And it's still one contiguous array, so it keeps every locality property from
lesson 04.

**Rust:**

```rust
use std::fmt;
use std::hint::black_box;
use std::time::{Duration, Instant};

// A dynamic array is Theta(n) to insert in the middle. A text editor inserts in
// the middle on EVERY KEYSTROKE. So how does one hold a 10 MB file and stay
// responsive?
//
// The GAP BUFFER: keep the array, but leave the free space in the MIDDLE,
// exactly where the cursor is. Then a keystroke is Theta(1) -- it just fills one
// slot of the gap. Moving the cursor is what costs, and it costs only the
// distance moved.
//
//   [ h e l l o _ _ _ _ _ _  w o r l d ]
//                ^^^^^^^^^^^
//               gap_start  gap_end
//
// This is a real structure -- Emacs is built on one -- and it is the cleanest
// example of the idea that you can move a cost somewhere the workload does not
// look.

struct GapBuffer {
    buf: Vec<u8>,
    gap_start: usize, // first free slot
    gap_end: usize,   // one past the last free slot
}

impl GapBuffer {
    fn new(gap: usize) -> Self {
        // The whole Vec is the gap to begin with. Its length (not capacity) is
        // what the gap arithmetic uses, so it is filled with zeros up front.
        GapBuffer {
            buf: vec![0; gap],
            gap_start: 0,
            gap_end: gap,
        }
    }

    fn len(&self) -> usize {
        self.buf.len() - (self.gap_end - self.gap_start)
    }

    /// grow doubles the buffer, placing a fresh gap at the cursor.
    fn grow(&mut self) {
        let new_gap = self.buf.len().max(8);
        let tail_len = self.buf.len() - self.gap_end;
        let mut bigger = vec![0u8; self.buf.len() + new_gap];
        bigger[..self.gap_start].copy_from_slice(&self.buf[..self.gap_start]);
        let tail_start = bigger.len() - tail_len;
        bigger[tail_start..].copy_from_slice(&self.buf[self.gap_end..]);
        self.buf = bigger; // the old Vec is dropped (freed) right here
        self.gap_end = tail_start;
    }

    /// insert writes one byte at the cursor. Theta(1) amortized -- no shifting at all.
    fn insert(&mut self, c: u8) {
        if self.gap_start == self.gap_end {
            self.grow();
        }
        self.buf[self.gap_start] = c;
        self.gap_start += 1;
    }

    /// delete removes the byte before the cursor. Theta(1): just widen the gap.
    fn delete(&mut self) {
        if self.gap_start > 0 {
            self.gap_start -= 1;
        }
    }

    /// move_to puts the cursor at position pos, moving the gap there.
    /// Theta(distance): the bytes the gap passes over are copied across it.
    /// (Go's version moves them one byte per loop turn; copy_within is the same
    /// copy done as one memmove -- the cost is still proportional to the distance.)
    fn move_to(&mut self, pos: usize) {
        if pos < self.gap_start {
            // move left: the bytes pos..gap_start slide to the far side of the gap
            let n = self.gap_start - pos;
            self.buf.copy_within(pos..self.gap_start, self.gap_end - n);
            self.gap_start -= n;
            self.gap_end -= n;
        } else if pos > self.gap_start {
            // move right: the bytes just past the gap slide to its near side
            let n = pos - self.gap_start;
            self.buf
                .copy_within(self.gap_end..self.gap_end + n, self.gap_start);
            self.gap_start += n;
            self.gap_end += n;
        }
    }

    /// debug renders the gap visibly.
    fn debug(&self) -> String {
        format!(
            "{}[{}]{}",
            String::from_utf8_lossy(&self.buf[..self.gap_start]),
            "_".repeat(self.gap_end - self.gap_start),
            String::from_utf8_lossy(&self.buf[self.gap_end..])
        )
    }
}

// Go's String() method: the text with the gap squeezed out. (ASCII only here;
// from_utf8_lossy would paper over a gap that landed inside a multibyte character.)
impl fmt::Display for GapBuffer {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(
            f,
            "{}{}",
            String::from_utf8_lossy(&self.buf[..self.gap_start]),
            String::from_utf8_lossy(&self.buf[self.gap_end..])
        )
    }
}

// --- the naive alternative: a plain Vec<u8> ---

struct PlainBuffer {
    buf: Vec<u8>,
}

impl PlainBuffer {
    fn insert_at(&mut self, pos: usize, c: u8) {
        // Vec::insert is exactly Go's append(0) + copy(buf[pos+1:], buf[pos:]) + store
        self.buf.insert(pos, c);
    }
}

// stand-in for testing.Benchmark / b.Loop: ns per call of `f`
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
    println!("watch the gap move as you type:");
    println!();
    let mut g = GapBuffer::new(4);
    for c in "hello".bytes() {
        g.insert(c);
    }
    println!(
        "  type 'hello'        {:<22} -> {:?}",
        g.debug(),
        g.to_string()
    );

    g.move_to(0);
    println!("  cursor to start     {:<22}", g.debug());

    for c in ">> ".bytes() {
        g.insert(c);
    }
    println!(
        "  type '>> '          {:<22} -> {:?}",
        g.debug(),
        g.to_string()
    );

    g.move_to(g.len());
    for c in "!".bytes() {
        g.insert(c);
    }
    println!(
        "  cursor to end, '!'  {:<22} -> {:?}",
        g.debug(),
        g.to_string()
    );

    g.delete();
    println!(
        "  backspace           {:<22} -> {:?}",
        g.debug(),
        g.to_string()
    );

    println!();
    println!("now the workload that matters: typing n characters at the FRONT of a");
    println!("document -- the worst case for a plain array, the best for a gap buffer.");
    println!();
    println!(
        "  {:>10} {:>18} {:>18} {:>12}",
        "n", "plain Vec<u8> ns", "gap buffer ns", "speedup"
    );

    // (n, plain ns, gap ns)
    let mut rows: Vec<(usize, f64, f64)> = Vec::new();
    for n in [1000usize, 4000, 16000] {
        let t_plain = ns_per_op(|| {
            let mut p = PlainBuffer { buf: Vec::new() };
            for _ in 0..n {
                p.insert_at(0, b'x'); // always at the front: shift everything
            }
            black_box(p.buf[0]);
        });
        let t_gap = ns_per_op(|| {
            let mut gb = GapBuffer::new(8);
            for _ in 0..n {
                gb.move_to(0); // the cursor is already there after the first move
                gb.insert(b'x');
            }
            black_box(gb.to_string());
        });
        println!(
            "  {n:>10} {t_plain:>18.0} {t_gap:>18.0} {:>11.1}x",
            t_plain / t_gap
        );
        rows.push((n, t_plain, t_gap));
    }

    // Printed from the rows above, so it cannot disagree with them.
    let (first, last) = (rows[0], rows[rows.len() - 1]);
    println!();
    let grow = last.0 / first.0;
    println!(
        "n grew {grow}x ({} -> {}); linear growth would be {grow}x, Theta(n^2) would be {}x.",
        first.0,
        last.0,
        grow * grow
    );
    println!(
        "The plain array took {:.0}x longer, the gap buffer {:.0}x:",
        last.1 / first.1,
        last.2 / first.2
    );
    println!("the plain array is Theta(n^2): every insert shifts everything after it.");
    println!("The gap buffer is Theta(n): the gap is already at the cursor, so each");
    println!("keystroke fills one slot.");

    println!();
    println!("what it costs, honestly:");
    println!();
    println!("  move_to is Theta(distance). Jumping from the top of a file to the");
    println!("  bottom copies the whole document. A gap buffer is a bet that edits");
    println!("  CLUSTER -- which, for a human typing, they overwhelmingly do.");
    println!();
    println!("  it is also still one contiguous array, so it keeps every locality");
    println!("  property from lesson 04. That is why editors preferred it to a");
    println!("  linked list of lines for decades.");
    println!();
    println!("the transferable idea: you cannot remove the Theta(n) shift from an");
    println!("array, but you CAN choose where it happens -- and put it somewhere");
    println!("the workload rarely looks.");
}
```

**Rust sample output:**

```
watch the gap move as you type:

  type 'hello'        hello[_______]         -> "hello"
  cursor to start     [_______]hello        
  type '>> '          >> [____]hello         -> ">> hello"
  cursor to end, '!'  >> hello![___]         -> ">> hello!"
  backspace           >> hello[____]         -> ">> hello"

now the workload that matters: typing n characters at the FRONT of a
document -- the worst case for a plain array, the best for a gap buffer.

           n   plain Vec<u8> ns      gap buffer ns      speedup
        1000               7449               1260         5.9x
        4000              81636               4298        19.0x
       16000            1180204              16292        72.4x

n grew 16x (1000 -> 16000); linear growth would be 16x, Theta(n^2) would be 256x.
The plain array took 158x longer, the gap buffer 13x:
the plain array is Theta(n^2): every insert shifts everything after it.
The gap buffer is Theta(n): the gap is already at the cursor, so each
keystroke fills one slot.

what it costs, honestly:

  move_to is Theta(distance). Jumping from the top of a file to the
  bottom copies the whole document. A gap buffer is a bet that edits
  CLUSTER -- which, for a human typing, they overwhelmingly do.

  it is also still one contiguous array, so it keeps every locality
  property from lesson 04. That is why editors preferred it to a
  linked list of lines for decades.

the transferable idea: you cannot remove the Theta(n) shift from an
array, but you CAN choose where it happens -- and put it somewhere
the workload rarely looks.
```

> **Go vs Rust:** The five lines that render the gap are byte-for-byte the same as Go's (they depend only on the algorithm); the timing table is a sample. The speedup grows about in proportion to n (roughly 6x, 19x, 73x for 4x steps in n; Go's sample: 5.1x, 13.7x, 50.4x), and at n=16,000 the plain array costs about the same as in Go (about 1.2 ms against 1.17 ms) while the gap buffer is faster (16-17 µs against 23 µs). `Vec::insert` is the plain buffer's `append` + `copy`, and `move_to` uses one `copy_within` memmove where Go loops byte by byte, which this benchmark never exercises because the cursor is already at 0.

**Complexity:** `Insert`/`Delete` at the cursor **Θ(1) amortized** · `MoveTo` Θ(distance) · the transferable idea: you can't remove the shift, but you can choose where it happens

---

## 14. The accidental quadratic

`🔴 hard` · *`slices.Delete` in a loop*

The most common accidental Θ(n²) in production Go, and it looks entirely reasonable — because the
Θ(n) is hidden inside a standard-library call.

**Steps:**

1. Remove a set of elements by calling `slices.Delete` per element.
2. Do the same in one pass with the write-index idiom.
3. Compare at four sizes and watch the ratio grow.

```go
package main

import (
	"fmt"
	"slices"
	"testing"
)

// The most common accidental Theta(n^2) in production Go, and it looks
// completely reasonable:
//
//   for _, id := range toRemove {
//       i := slices.Index(items, id)
//       items = slices.Delete(items, i, i+1)
//   }
//
// Each Delete is Theta(n). Doing it k times is Theta(n*k) -- which is Theta(n^2)
// when you are removing a constant fraction of the collection.
//
// The fix is example 3's write-index idiom: ONE pass, Theta(n + k) total.

var sink []int

// removeRepeated: delete them one at a time. Correct, and quadratic.
func removeRepeated(xs []int, drop map[int]struct{}) []int {
	for i := 0; i < len(xs); {
		if _, gone := drop[xs[i]]; gone {
			xs = slices.Delete(xs, i, i+1) // Theta(n) EVERY time
			continue                       // do not advance: the tail slid left
		}
		i++
	}
	return xs
}

// removeOnePass: the write-index filter. One pass, no shifting.
func removeOnePass(xs []int, drop map[int]struct{}) []int {
	write := 0
	for _, v := range xs {
		if _, gone := drop[v]; gone {
			continue
		}
		xs[write] = v
		write++
	}
	return xs[:write]
}

// removeDeleteFunc: the stdlib's version of the same one-pass idea.
func removeDeleteFunc(xs []int, drop map[int]struct{}) []int {
	return slices.DeleteFunc(xs, func(v int) bool {
		_, gone := drop[v]
		return gone
	})
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
	// First: all three agree.
	src := []int{1, 2, 3, 4, 5, 6, 7, 8}
	drop := map[int]struct{}{2: {}, 4: {}, 6: {}}
	fmt.Println("removing {2,4,6} from [1 2 3 4 5 6 7 8]:")
	fmt.Println()
	fmt.Printf("  repeated Delete   %v\n", removeRepeated(slices.Clone(src), drop))
	fmt.Printf("  one pass          %v\n", removeOnePass(slices.Clone(src), drop))
	fmt.Printf("  slices.DeleteFunc %v\n", removeDeleteFunc(slices.Clone(src), drop))

	fmt.Println()
	fmt.Println("now remove HALF the elements from a slice of n:")
	fmt.Println()
	fmt.Printf("  %10s %18s %16s %18s %12s\n",
		"n", "repeated Delete", "one pass", "slices.DeleteFunc", "ratio")

	for _, n := range []int{1_000, 4_000, 16_000, 64_000} {
		base := make([]int, n)
		d := make(map[int]struct{}, n/2)
		for i := range base {
			base[i] = i
			if i%2 == 0 {
				d[i] = struct{}{}
			}
		}

		tRep := nsPerOp(func() { sink = removeRepeated(slices.Clone(base), d) })
		tOne := nsPerOp(func() { sink = removeOnePass(slices.Clone(base), d) })
		tStd := nsPerOp(func() { sink = removeDeleteFunc(slices.Clone(base), d) })

		fmt.Printf("  %10d %18.0f %16.0f %18.0f %11.0fx\n", n, tRep, tOne, tStd, tRep/tOne)
	}

	fmt.Println()
	fmt.Println("the ratio column grows with n, which is the signature of a whole")
	fmt.Println("complexity class difference rather than a constant factor:")
	fmt.Println()
	fmt.Println("  repeated Delete   Theta(n * k)  -- Theta(n^2) when k is a fraction of n")
	fmt.Println("  one pass          Theta(n)")
	fmt.Println()
	fmt.Println("note the middle and right columns are the same algorithm: slices.DeleteFunc")
	fmt.Println("IS the write-index idiom, written once in the standard library. Reaching")
	fmt.Println("for it is both shorter and correct -- but only if you know that")
	fmt.Println("slices.Delete in a loop is the thing it replaces.")

	fmt.Println()
	fmt.Println("this is lesson 03's 'quadratic hiding in a single loop' with a")
	fmt.Println("standard-library function playing the part of the hidden Theta(n).")
	fmt.Println("Calling stdlib does not make an operation cheap; it makes it correct.")
	fmt.Println()
	fmt.Println("the rule: removing ONE element -> Delete. Removing MANY -> one pass.")
	fmt.Println("If the word 'loop' and the word 'Delete' appear together, look again.")
}
```

**Sample output:**

```
removing {2,4,6} from [1 2 3 4 5 6 7 8]:

  repeated Delete   [1 3 5 7 8]
  one pass          [1 3 5 7 8]
  slices.DeleteFunc [1 3 5 7 8]

now remove HALF the elements from a slice of n:

           n    repeated Delete         one pass  slices.DeleteFunc        ratio
        1000              24806             4050               4535           6x
        4000             327836            18695              19829          18x
       16000            5061064            80811              89852          63x
       64000          121106148           335216             389893         361x

the ratio column grows with n, which is the signature of a whole
complexity class difference rather than a constant factor:

  repeated Delete   Theta(n * k)  -- Theta(n^2) when k is a fraction of n
  one pass          Theta(n)

note the middle and right columns are the same algorithm: slices.DeleteFunc
IS the write-index idiom, written once in the standard library. Reaching
for it is both shorter and correct -- but only if you know that
slices.Delete in a loop is the thing it replaces.

this is lesson 03's 'quadratic hiding in a single loop' with a
standard-library function playing the part of the hidden Theta(n).
Calling stdlib does not make an operation cheap; it makes it correct.

the rule: removing ONE element -> Delete. Removing MANY -> one pass.
If the word 'loop' and the word 'Delete' appear together, look again.
```

**6× → 17× → 64× → 355×.** The ratio growing with n is the signature of a complexity-class
difference, not a constant factor.

Note the middle and right columns are the same algorithm: `slices.DeleteFunc` **is** the write-index
idiom, written once in the standard library. This is lesson 03's "quadratic hiding in a single loop"
with a stdlib function playing the part of the hidden Θ(n).

**Rust:**

```rust
use std::collections::HashSet;
use std::hint::black_box;
use std::time::{Duration, Instant};

// The most common accidental Theta(n^2) in production Go has a Rust twin, and
// it looks just as reasonable:
//
//   for id in &to_remove {
//       let i = items.iter().position(|x| x == id).unwrap();
//       items.remove(i);
//   }
//
// Each Vec::remove is Theta(n): it shifts the whole tail one slot left. Doing it
// k times is Theta(n*k) -- which is Theta(n^2) when you are removing a constant
// fraction of the collection.
//
// The fix is example 3's write-index idiom: ONE pass, Theta(n + k) total.

/// Go's %v of a slice: [1 2 3]
fn list(xs: &[i64]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// remove_repeated: delete them one at a time. Correct, and quadratic.
fn remove_repeated(mut xs: Vec<i64>, drop: &HashSet<i64>) -> Vec<i64> {
    let mut i = 0;
    while i < xs.len() {
        if drop.contains(&xs[i]) {
            xs.remove(i); // Theta(n) EVERY time; do not advance: the tail slid left
        } else {
            i += 1;
        }
    }
    xs
}

/// remove_one_pass: the write-index filter. One pass, no shifting.
fn remove_one_pass(mut xs: Vec<i64>, drop: &HashSet<i64>) -> Vec<i64> {
    let mut write = 0;
    // `for &v in &xs` would hold a shared borrow of xs for the whole loop, and
    // `xs[write] = v` needs a mutable one -- the borrow checker refuses. So we
    // walk by index instead; i64 is Copy, so reading xs[read] out is free.
    for read in 0..xs.len() {
        let v = xs[read];
        if drop.contains(&v) {
            continue;
        }
        xs[write] = v;
        write += 1;
    }
    xs.truncate(write); // Go's xs[:write]
    xs
}

/// remove_retain: the standard library's version of the same one-pass idea.
/// Note the predicate is inverted: retain keeps what the closure accepts.
fn remove_retain(mut xs: Vec<i64>, drop: &HashSet<i64>) -> Vec<i64> {
    xs.retain(|v| !drop.contains(v));
    xs
}

// stand-in for testing.Benchmark / b.Loop: ns per call of `f`
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
    // First: all three agree.
    let src: Vec<i64> = vec![1, 2, 3, 4, 5, 6, 7, 8];
    let drop: HashSet<i64> = HashSet::from([2, 4, 6]);
    println!("removing {{2,4,6}} from [1 2 3 4 5 6 7 8]:");
    println!();
    println!(
        "  repeated remove   {}",
        list(&remove_repeated(src.clone(), &drop))
    );
    println!(
        "  one pass          {}",
        list(&remove_one_pass(src.clone(), &drop))
    );
    println!(
        "  Vec::retain       {}",
        list(&remove_retain(src.clone(), &drop))
    );

    println!();
    println!("now remove HALF the elements from a Vec of n:");
    println!();
    println!(
        "  {:>10} {:>18} {:>16} {:>18} {:>12}",
        "n", "repeated remove", "one pass", "Vec::retain", "ratio"
    );

    // (n, repeated ns, one-pass ns, retain ns)
    let mut rows: Vec<(usize, f64, f64, f64)> = Vec::new();
    for n in [1_000usize, 4_000, 16_000, 64_000] {
        let base: Vec<i64> = (0..n as i64).collect();
        let d: HashSet<i64> = (0..n as i64).filter(|i| i % 2 == 0).collect();

        let t_rep = ns_per_op(|| {
            black_box(remove_repeated(base.clone(), &d));
        });
        let t_one = ns_per_op(|| {
            black_box(remove_one_pass(base.clone(), &d));
        });
        let t_std = ns_per_op(|| {
            black_box(remove_retain(base.clone(), &d));
        });

        println!(
            "  {n:>10} {t_rep:>18.0} {t_one:>16.0} {t_std:>18.0} {:>11.0}x",
            t_rep / t_one
        );
        rows.push((n, t_rep, t_one, t_std));
    }

    // Everything printed below is computed from the rows above, so it cannot
    // disagree with them.
    let ratios: Vec<f64> = rows.iter().map(|r| r.1 / r.2).collect();
    let growing = ratios.windows(2).all(|w| w[1] > w[0]);
    let (first, last) = (rows[0], rows[rows.len() - 1]);
    let grow = last.0 / first.0;

    println!();
    if growing {
        println!("the ratio column grows with n at every step, which is the signature of a whole");
        println!("complexity class difference rather than a constant factor:");
    } else {
        println!("the ratio column does NOT grow at every step in this run (noise):");
    }
    println!();
    println!(
        "  n grew {grow}x ({} -> {}); linear growth would be {grow}x, Theta(n^2) {}x:",
        first.0,
        last.0,
        grow * grow
    );
    println!("    repeated remove   grew {:.0}x", last.1 / first.1);
    println!("    one pass          grew {:.0}x", last.2 / first.2);
    if last.1 / first.1 > (grow * grow) as f64 {
        println!("  (repeated remove grew MORE than Theta(n^2) predicts: the longer shifts also");
        println!("  stop fitting in cache -- a plausible mechanism, not measured here.)");
    }
    println!();
    println!("  repeated remove   Theta(n * k)  -- Theta(n^2) when k is a fraction of n");
    println!("  one pass          Theta(n)");

    let retain_vs_hand: Vec<f64> = rows.iter().map(|r| r.3 / r.2).collect();
    let lo = retain_vs_hand.iter().copied().fold(f64::MAX, f64::min);
    let hi = retain_vs_hand.iter().copied().fold(0.0, f64::max);
    println!();
    println!("the middle and right columns are the same algorithm: Vec::retain IS the");
    println!("write-index idiom, written once in the standard library. Across the four");
    println!("sizes, retain took between {lo:.2}x and {hi:.2}x of the hand-written pass.");
    println!("Reaching for it is both shorter and correct -- but only if you know that");
    println!("remove in a loop is the thing it replaces.");

    println!();
    println!("this is lesson 03's 'quadratic hiding in a single loop' with a");
    println!("standard-library function playing the part of the hidden Theta(n).");
    println!("Calling stdlib does not make an operation cheap; it makes it correct.");
    println!();
    println!("the rule: removing ONE element -> remove. Removing MANY -> one pass.");
    println!("If the word 'loop' and the word 'remove' appear together, look again.");
    println!("(If order does not matter, swap_remove is Theta(1) per element -- but it");
    println!("moves the LAST element into the hole, so it is a different program.)");
}
```

**Rust sample output:**

```
removing {2,4,6} from [1 2 3 4 5 6 7 8]:

  repeated remove   [1 3 5 7 8]
  one pass          [1 3 5 7 8]
  Vec::retain       [1 3 5 7 8]

now remove HALF the elements from a Vec of n:

           n    repeated remove         one pass        Vec::retain        ratio
        1000              23914             4192               3848           6x
        4000             349585            17171              16384          20x
       16000            5101735            73256              66332          70x
       64000          124112208           308651             278962         402x

the ratio column grows with n at every step, which is the signature of a whole
complexity class difference rather than a constant factor:

  n grew 64x (1000 -> 64000); linear growth would be 64x, Theta(n^2) 4096x:
    repeated remove   grew 5190x
    one pass          grew 74x
  (repeated remove grew MORE than Theta(n^2) predicts: the longer shifts also
  stop fitting in cache -- a plausible mechanism, not measured here.)

  repeated remove   Theta(n * k)  -- Theta(n^2) when k is a fraction of n
  one pass          Theta(n)

the middle and right columns are the same algorithm: Vec::retain IS the
write-index idiom, written once in the standard library. Across the four
sizes, retain took between 0.90x and 0.95x of the hand-written pass.
Reaching for it is both shorter and correct -- but only if you know that
remove in a loop is the thing it replaces.

this is lesson 03's 'quadratic hiding in a single loop' with a
standard-library function playing the part of the hidden Theta(n).
Calling stdlib does not make an operation cheap; it makes it correct.

the rule: removing ONE element -> remove. Removing MANY -> one pass.
If the word 'loop' and the word 'remove' appear together, look again.
(If order does not matter, swap_remove is Theta(1) per element -- but it
moves the LAST element into the hole, so it is a different program.)
```

> **Go vs Rust:** `Vec::remove` is the twin of `slices.Delete` and `Vec::retain` of `slices.DeleteFunc`; the three removed lists are identical to Go's and the ratio column lands close to Go's (Go: 6x, 18x, 63x, 361x), because both languages pay the same memmove. The write-index loop cannot be spelled the Go way: `for &v in &xs` holds a shared borrow of `xs` while `xs[write] = v` needs a mutable one (error E0502), so the Rust version walks by index. `retain` stayed within roughly 0.7x-1.1x of the hand-written pass, so using it costs nothing.

**Complexity:** repeated `Delete` **Θ(n·k)** · one pass **Θ(n)** · rule: removing one → `Delete`; removing many → one pass

---

## 15. Partition, and the invariant that makes it work

`🔴 hard` · *The engine inside quicksort*

Rearrange in place so everything matching a predicate comes first. Three versions, each maintaining a
stated invariant — and the third handles duplicates in a way that matters enormously later.

**Steps:**

1. Two-way partition with the write-index idiom, counting swaps.
2. Hoare's two-pointer version, which only swaps genuinely misplaced pairs.
3. The Dutch national flag: three regions, one pass.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
)

// PARTITION: rearrange a slice in place so everything satisfying a predicate
// comes first. One pass, no allocation -- and the engine inside quicksort,
// quickselect, and every "separate the wheat from the chaff" problem.
//
// Three versions, in increasing order of cleverness.

var swaps, comparisons int

func reset() { swaps, comparisons = 0, 0 }

func swap(xs []int, i, j int) {
	if i != j {
		swaps++
	}
	xs[i], xs[j] = xs[j], xs[i]
}

// --- 1. two-way partition: the write-index idiom with a swap ---
//
// Returns the boundary: xs[:b] satisfies the predicate, xs[b:] does not.
func partition(xs []int, pred func(int) bool) int {
	write := 0
	for read := 0; read < len(xs); read++ {
		comparisons++
		if pred(xs[read]) {
			swap(xs, write, read)
			write++
		}
	}
	return write
}

// --- 2. Hoare's two-pointer partition: fewer swaps ---
//
// Walk in from both ends and swap the pairs that are on the wrong side. It only
// swaps elements that are ACTUALLY misplaced, where the write-index version
// swaps on every match.
func partitionHoare(xs []int, pred func(int) bool) int {
	i, j := 0, len(xs)-1
	for {
		for i <= j {
			comparisons++
			if !pred(xs[i]) {
				break
			}
			i++
		}
		for i <= j {
			comparisons++
			if pred(xs[j]) {
				break
			}
			j--
		}
		if i > j {
			return i
		}
		swap(xs, i, j)
		i++
		j--
	}
}

// --- 3. Dutch national flag: THREE-way partition in one pass ---
//
// Named for Dijkstra's problem of sorting red/white/blue stripes. Returns the
// two boundaries: xs[:lo] < pivot, xs[lo:hi] == pivot, xs[hi:] > pivot.
//
// The invariant is the whole trick. At every step:
//
//	[ < pivot | == pivot | UNKNOWN | > pivot ]
//	 0       lo        mid       hi        n
//
// mid walks forward; lo and hi close in. The UNKNOWN region shrinks to nothing.
func dutchFlag(xs []int, pivot int) (lo, hi int) {
	lo, mid, hi := 0, 0, len(xs)
	for mid < hi {
		comparisons++
		switch {
		case xs[mid] < pivot:
			swap(xs, lo, mid)
			lo++
			mid++
		case xs[mid] > pivot:
			hi--
			swap(xs, mid, hi)
			// do NOT advance mid: the value swapped in is unexamined
		default:
			mid++
		}
	}
	return lo, hi
}

func isEven(v int) bool { return v%2 == 0 }

func main() {
	fmt.Println("two-way partition -- evens first:")
	fmt.Println()
	for _, name := range []string{"write-index", "Hoare"} {
		xs := []int{1, 2, 3, 4, 5, 6, 7, 8}
		reset()
		var b int
		if name == "write-index" {
			b = partition(xs, isEven)
		} else {
			b = partitionHoare(xs, isEven)
		}
		fmt.Printf("  %-14s %v  boundary=%d  swaps=%d comparisons=%d\n",
			name, xs, b, swaps, comparisons)
	}
	fmt.Println()
	fmt.Println("  both are correct and neither preserves the original order.")
	fmt.Println("  Hoare does fewer swaps -- it only touches genuinely misplaced pairs.")

	fmt.Println()
	fmt.Println("swaps over 10,000 random elements:")
	fmt.Println()
	rng := rand.New(rand.NewPCG(1, 2))
	base := make([]int, 10_000)
	for i := range base {
		base[i] = rng.IntN(1000)
	}
	reset()
	partition(slices.Clone(base), isEven)
	wSwaps, wComp := swaps, comparisons
	reset()
	partitionHoare(slices.Clone(base), isEven)
	fmt.Printf("  write-index   %6d swaps, %6d comparisons\n", wSwaps, wComp)
	fmt.Printf("  Hoare         %6d swaps, %6d comparisons\n", swaps, comparisons)

	fmt.Println()
	fmt.Println("three-way (Dutch national flag), pivot = 5:")
	fmt.Println()
	xs := []int{5, 3, 8, 5, 1, 9, 5, 2, 7}
	fmt.Printf("  before        %v\n", xs)
	lo, hi := dutchFlag(xs, 5)
	fmt.Printf("  after         %v\n", xs)
	fmt.Printf("  < 5           %v\n", xs[:lo])
	fmt.Printf("  == 5          %v\n", xs[lo:hi])
	fmt.Printf("  > 5           %v\n", xs[hi:])

	fmt.Println()
	fmt.Println("why the three-way version matters, in one number:")
	fmt.Println()
	dup := make([]int, 100_000)
	for i := range dup {
		dup[i] = rng.IntN(3) // only three distinct values
	}
	reset()
	dutchFlag(slices.Clone(dup), 1)
	fmt.Printf("  100,000 elements with 3 distinct values: %d comparisons, one pass\n", comparisons)
	fmt.Println()
	fmt.Println("  a two-way partition would put all the ==pivot elements on one side")
	fmt.Println("  and quicksort would then recurse on them again, and again. With many")
	fmt.Println("  duplicates that degenerates toward Theta(n^2). Three-way finishes the")
	fmt.Println("  equal block in this pass and never looks at it again -- which is why")
	fmt.Println("  Go's pdqsort uses a three-way partition (lesson 13).")

	fmt.Println()
	fmt.Println("the shared skeleton behind all three:")
	fmt.Println()
	fmt.Println("  maintain an INVARIANT about regions of the slice, and move the")
	fmt.Println("  boundaries so the unknown region shrinks to nothing. Every one is")
	fmt.Println("  Theta(n) time and Theta(1) space, and none of them allocates.")
	fmt.Println()
	fmt.Println("  the discipline that makes it work is writing the invariant down")
	fmt.Println("  FIRST. The Dutch-flag `do not advance mid` is impossible to get")
	fmt.Println("  right by intuition and obvious once the invariant is stated.")
}
```

**Output:**

```
two-way partition -- evens first:

  write-index    [2 4 6 8 5 3 7 1]  boundary=4  swaps=4 comparisons=8
  Hoare          [8 2 6 4 5 3 7 1]  boundary=4  swaps=2 comparisons=9

  both are correct and neither preserves the original order.
  Hoare does fewer swaps -- it only touches genuinely misplaced pairs.

swaps over 10,000 random elements:

  write-index     4976 swaps,  10000 comparisons
  Hoare           2524 swaps,  10000 comparisons

three-way (Dutch national flag), pivot = 5:

  before        [5 3 8 5 1 9 5 2 7]
  after         [3 2 1 5 5 5 9 7 8]
  < 5           [3 2 1]
  == 5          [5 5 5]
  > 5           [9 7 8]

why the three-way version matters, in one number:

  100,000 elements with 3 distinct values: 100000 comparisons, one pass

  a two-way partition would put all the ==pivot elements on one side
  and quicksort would then recurse on them again, and again. With many
  duplicates that degenerates toward Theta(n^2). Three-way finishes the
  equal block in this pass and never looks at it again -- which is why
  Go's pdqsort uses a three-way partition (lesson 13).

the shared skeleton behind all three:

  maintain an INVARIANT about regions of the slice, and move the
  boundaries so the unknown region shrinks to nothing. Every one is
  Theta(n) time and Theta(1) space, and none of them allocates.

  the discipline that makes it work is writing the invariant down
  FIRST. The Dutch-flag `do not advance mid` is impossible to get
  right by intuition and obvious once the invariant is stated.
```

Hoare does **half the swaps** (2524 vs 4976) for the same comparisons, because it only touches pairs
that are actually on the wrong side.

The three-way version is the one that matters for lesson 13: a two-way partition puts all the
`== pivot` elements on one side, and quicksort then recurses on them again and again — degenerating
toward Θ(n²) with many duplicates. Three-way finishes the equal block in this pass and never revisits
it, which is why Go's pdqsort uses one.

The `do not advance mid` case is impossible to get right by intuition and obvious once the invariant
is written down. Write the invariant first.

**Rust:**

```rust
use std::cmp::Ordering;
use std::fmt::Display;

// PARTITION: rearrange a slice in place so everything satisfying a predicate
// comes first. One pass, no allocation -- and the engine inside quicksort,
// quickselect, and every "separate the wheat from the chaff" problem.
//
// Three versions, in increasing order of cleverness.

// Go keeps `swaps` and `comparisons` in package-level variables and resets them
// between runs. Rust has no mutable globals (short of atomics or `unsafe`), so
// the counters travel with the call: one `Stats` per measurement, passed as
// `&mut`. `Stats::default()` is Go's reset().
#[derive(Default)]
struct Stats {
    swaps: usize,
    comparisons: usize,
}

impl Stats {
    fn swap(&mut self, xs: &mut [i64], i: usize, j: usize) {
        if i != j {
            self.swaps += 1;
        }
        xs.swap(i, j);
    }
}

// Go's `func(int) bool`. A plain fn pointer keeps the two algorithms in one
// table below; a generic `impl Fn` would be inlined (faster) but could not.
type Pred = fn(i64) -> bool;

// Both two-way versions have this shape, so they can share one table.
type TwoWay = fn(&mut [i64], Pred, &mut Stats) -> usize;

// --- 1. two-way partition: the write-index idiom with a swap ---
//
// Returns the boundary: xs[..b] satisfies the predicate, xs[b..] does not.
fn partition(xs: &mut [i64], pred: Pred, st: &mut Stats) -> usize {
    let mut write = 0;
    for read in 0..xs.len() {
        st.comparisons += 1;
        if pred(xs[read]) {
            st.swap(xs, write, read);
            write += 1;
        }
    }
    write
}

// --- 2. Hoare's two-pointer partition: fewer swaps ---
//
// Walk in from both ends and swap the pairs that are on the wrong side. It only
// swaps elements that are ACTUALLY misplaced, where the write-index version
// swaps on every match.
//
// Go's right-hand pointer `j` starts at len-1 and can end at -1, which a usize
// cannot hold. So `end` is the EXCLUSIVE end (end == j+1): the unexamined
// region is xs[i..end], and "i <= j" becomes "i < end". Same steps, same counts.
fn partition_hoare(xs: &mut [i64], pred: Pred, st: &mut Stats) -> usize {
    let (mut i, mut end) = (0, xs.len());
    loop {
        while i < end {
            st.comparisons += 1;
            if !pred(xs[i]) {
                break;
            }
            i += 1;
        }
        while i < end {
            st.comparisons += 1;
            if pred(xs[end - 1]) {
                break;
            }
            end -= 1;
        }
        if i >= end {
            return i;
        }
        st.swap(xs, i, end - 1);
        i += 1;
        end -= 1;
    }
}

// --- 3. Dutch national flag: THREE-way partition in one pass ---
//
// Named for Dijkstra's problem of sorting red/white/blue stripes. Returns the
// two boundaries: xs[..lo] < pivot, xs[lo..hi] == pivot, xs[hi..] > pivot.
//
// The invariant is the whole trick. At every step:
//
//   [ < pivot | == pivot | UNKNOWN | > pivot ]
//    0       lo        mid       hi        n
//
// mid walks forward; lo and hi close in. The UNKNOWN region shrinks to nothing.
fn dutch_flag(xs: &mut [i64], pivot: i64, st: &mut Stats) -> (usize, usize) {
    let (mut lo, mut mid, mut hi) = (0, 0, xs.len());
    while mid < hi {
        st.comparisons += 1;
        // Go's `switch { case <: case >: default: }` is a match on Ordering.
        match xs[mid].cmp(&pivot) {
            Ordering::Less => {
                st.swap(xs, lo, mid);
                lo += 1;
                mid += 1;
            }
            Ordering::Greater => {
                hi -= 1;
                st.swap(xs, mid, hi);
                // do NOT advance mid: the value swapped in is unexamined
            }
            Ordering::Equal => mid += 1,
        }
    }
    (lo, hi)
}

fn is_even(v: i64) -> bool {
    v % 2 == 0
}

// ---- stand-ins for what Go's stdlib gives for free ----

/// Go prints a slice as [1 2 3]; Rust's {:?} prints [1, 2, 3]. Format by hand
/// so the output reads the same in both languages.
fn go_slice<T: Display>(xs: &[T]) -> String {
    let items: Vec<String> = xs.iter().map(T::to_string).collect();
    format!("[{}]", items.join(" "))
}

/// A port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its IntN.
/// With the same generator and seed, the 10,000 and 100,000 "random" elements
/// below are exactly the ones the Go program sees, so every count can match.
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
    println!("two-way partition -- evens first:");
    println!();
    let algorithms: [(&str, TwoWay); 2] = [("write-index", partition), ("Hoare", partition_hoare)];
    for (name, algorithm) in algorithms {
        let mut xs = [1, 2, 3, 4, 5, 6, 7, 8]; // an array: [i64; 8], no heap
        let mut st = Stats::default();
        let b = algorithm(&mut xs, is_even, &mut st);
        println!(
            "  {:<14} {}  boundary={}  swaps={} comparisons={}",
            name,
            go_slice(&xs),
            b,
            st.swaps,
            st.comparisons
        );
    }
    println!();
    println!("  both are correct and neither preserves the original order.");
    println!("  Hoare does fewer swaps -- it only touches genuinely misplaced pairs.");

    println!();
    println!("swaps over 10,000 random elements:");
    println!();
    let mut rng = Pcg::new(1, 2);
    let base: Vec<i64> = (0..10_000).map(|_| rng.int_n(1000) as i64).collect();
    let mut w = Stats::default();
    partition(&mut base.clone(), is_even, &mut w);
    let mut h = Stats::default();
    partition_hoare(&mut base.clone(), is_even, &mut h);
    println!(
        "  write-index   {:>6} swaps, {:>6} comparisons",
        w.swaps, w.comparisons
    );
    println!(
        "  Hoare         {:>6} swaps, {:>6} comparisons",
        h.swaps, h.comparisons
    );

    println!();
    println!("three-way (Dutch national flag), pivot = 5:");
    println!();
    let mut xs = [5, 3, 8, 5, 1, 9, 5, 2, 7];
    println!("  before        {}", go_slice(&xs));
    let (lo, hi) = dutch_flag(&mut xs, 5, &mut Stats::default());
    println!("  after         {}", go_slice(&xs));
    println!("  < 5           {}", go_slice(&xs[..lo]));
    println!("  == 5          {}", go_slice(&xs[lo..hi]));
    println!("  > 5           {}", go_slice(&xs[hi..]));

    println!();
    println!("why the three-way version matters, in one number:");
    println!();
    let dup: Vec<i64> = (0..100_000).map(|_| rng.int_n(3) as i64).collect(); // only three distinct values
    let mut st = Stats::default();
    dutch_flag(&mut dup.clone(), 1, &mut st);
    println!(
        "  100,000 elements with 3 distinct values: {} comparisons, one pass",
        st.comparisons
    );
    println!();
    println!("  a two-way partition would put all the ==pivot elements on one side");
    println!("  and quicksort would then recurse on them again, and again. With many");
    println!("  duplicates that degenerates toward Theta(n^2). Three-way finishes the");
    println!("  equal block in this pass and never looks at it again -- which is why");
    println!("  Go's pdqsort uses a three-way partition (lesson 13).");

    println!();
    println!("the shared skeleton behind all three:");
    println!();
    println!("  maintain an INVARIANT about regions of the slice, and move the");
    println!("  boundaries so the unknown region shrinks to nothing. Every one is");
    println!("  Theta(n) time and Theta(1) space, and none of them allocates.");
    println!();
    println!("  the discipline that makes it work is writing the invariant down");
    println!("  FIRST. The Dutch-flag `do not advance mid` is impossible to get");
    println!("  right by intuition and obvious once the invariant is stated.");

    // ---- Rust only: the same 100,000 duplicates, through std's sort ----
    //
    // Go's text above says pdqsort treats duplicates specially. Count what
    // Rust's own `sort_unstable` does instead of assuming: a comparator that
    // counts its calls, on the 3-distinct-value input and on an input that is
    // almost all distinct. (Comparison counts are deterministic for a fixed
    // input and a fixed std version; the input is the Go program's.)
    println!();
    println!("Rust only: comparisons made by sort_unstable on 100,000 elements:");
    println!();
    let count_sort = |data: &[i64]| {
        let mut v = data.to_vec();
        let mut calls = 0usize;
        v.sort_unstable_by(|a, b| {
            calls += 1;
            a.cmp(b)
        });
        calls
    };
    let distinct: Vec<i64> = (0..100_000).map(|_| rng.int_n(1 << 40) as i64).collect();
    let n = dup.len() as f64;
    println!("  3 distinct values   {:>9}", count_sort(&dup));
    println!("  almost all distinct {:>9}", count_sort(&distinct));
    println!("  n * log2(n)         {:>9.0}", n * n.log2());
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O ex15.rs -o ex15 && ./ex15
```

**Rust output:**

```
two-way partition -- evens first:

  write-index    [2 4 6 8 5 3 7 1]  boundary=4  swaps=4 comparisons=8
  Hoare          [8 2 6 4 5 3 7 1]  boundary=4  swaps=2 comparisons=9

  both are correct and neither preserves the original order.
  Hoare does fewer swaps -- it only touches genuinely misplaced pairs.

swaps over 10,000 random elements:

  write-index     4976 swaps,  10000 comparisons
  Hoare           2524 swaps,  10000 comparisons

three-way (Dutch national flag), pivot = 5:

  before        [5 3 8 5 1 9 5 2 7]
  after         [3 2 1 5 5 5 9 7 8]
  < 5           [3 2 1]
  == 5          [5 5 5]
  > 5           [9 7 8]

why the three-way version matters, in one number:

  100,000 elements with 3 distinct values: 100000 comparisons, one pass

  a two-way partition would put all the ==pivot elements on one side
  and quicksort would then recurse on them again, and again. With many
  duplicates that degenerates toward Theta(n^2). Three-way finishes the
  equal block in this pass and never looks at it again -- which is why
  Go's pdqsort uses a three-way partition (lesson 13).

the shared skeleton behind all three:

  maintain an INVARIANT about regions of the slice, and move the
  boundaries so the unknown region shrinks to nothing. Every one is
  Theta(n) time and Theta(1) space, and none of them allocates.

  the discipline that makes it work is writing the invariant down
  FIRST. The Dutch-flag `do not advance mid` is impossible to get
  right by intuition and obvious once the invariant is stated.

Rust only: comparisons made by sort_unstable on 100,000 elements:

  3 distinct values      302372
  almost all distinct   1724525
  n * log2(n)           1660964
```

> **Go vs Rust:** Every number above the last section is identical to Go's (swaps 4976 vs 2524, 100,000 comparisons for the flag), because the program is deterministic once the input is: a port of Go's `math/rand/v2` PCG generates the same 10,000 and 100,000 elements. The source differs in three places: the counters travel as a `&mut Stats` argument because Rust has no mutable globals, Hoare's right-hand pointer is an exclusive `end` because a `usize` cannot reach Go's `j = -1`, and Go's `switch` is a `match` on `Ordering`.
> The last section, "Rust only", is measured, not copied: `sort_unstable` needed 302,372 comparisons for the 100,000 three-valued elements against 1,724,525 for random distinct ones (n log2 n = 1,660,964), so std also stops re-sorting the duplicates. Its source (`core/src/slice/sort/unstable/quicksort.rs`, read here, not instrumented) does it with a second two-way pass over the equal-to-pivot elements when the pivot matches its ancestor, not with a one-pass Dutch flag; Go's `partitionEqual` in `slices/zsortordered.go` has the same shape, which makes the Go prose's "pdqsort uses a three-way partition" a simplification.

**Complexity:** all three Θ(n) time, **Θ(1) space**, no allocation · none preserves the original order

---

## 16. Capstone: `Vector[T]`, all three passes

`🔴 hard` · *Build it · Prove it · Measure it* · `go test`

The first structure in this plan, finished properly. This is what the three-pass method looks like in
practice, and the shape every later lesson follows.

**Pass 2 (prove it) has four layers**, each catching what the others can't:

| Layer | Catches |
|---|---|
| a table | specific known-answer cases, and the panics |
| invariants | "no removed element is still reachable" — a property no table case would check |
| a **model oracle** | 20,000 random operations against a plain `[]T`. This is the layer that finds real bugs |
| budgets & gates | allocation count, geometric growth, and no shrink thrashing |

**`vector.go`** — the structure, with its invariant written down

```go
package vector

// Vector[T] is the dynamic array from examples 7-9, finished: generic, with the
// hysteresis shrink policy, the pointer hygiene, and a stated invariant.
//
// INVARIANT, true before and after every exported method:
//
//	0 <= length <= len(data)
//	data[length:] holds only zero values      (so no removed T is kept alive)
//
// The zero Vector[T] is a valid empty vector -- no constructor required.
type Vector[T any] struct {
	data   []T
	length int
}

const (
	minCap        = 4
	growthFactor  = 2
	shrinkTrigger = 4 // shrink when length*4 <= cap ... (example 8)
	shrinkFactor  = 2 // ... and only halve, never shrink to fit
)

func (v *Vector[T]) Len() int { return v.length }
func (v *Vector[T]) Cap() int { return len(v.data) }

// At returns the element at i. Theta(1).
func (v *Vector[T]) At(i int) T {
	v.mustIndex(i)
	return v.data[i]
}

// Set overwrites the element at i. Theta(1).
func (v *Vector[T]) Set(i int, x T) {
	v.mustIndex(i)
	v.data[i] = x
}

func (v *Vector[T]) mustIndex(i int) {
	if i < 0 || i >= v.length {
		panic("vector: index out of range")
	}
}

func (v *Vector[T]) resize(newCap int) {
	if newCap < minCap {
		newCap = minCap
	}
	bigger := make([]T, newCap)
	copy(bigger, v.data[:v.length])
	v.data = bigger
}

// Push appends x. Theta(1) amortized.
func (v *Vector[T]) Push(x T) {
	if v.length == len(v.data) {
		v.resize(max(len(v.data)*growthFactor, minCap))
	}
	v.data[v.length] = x
	v.length++
}

// Pop removes and returns the last element. Theta(1) amortized.
func (v *Vector[T]) Pop() (T, bool) {
	var zero T
	if v.length == 0 {
		return zero, false
	}
	v.length--
	x := v.data[v.length]
	v.data[v.length] = zero // invariant: nothing past length holds a live T
	v.maybeShrink()
	return x, true
}

// Insert places x at index i, shifting the tail right. Theta(n).
func (v *Vector[T]) Insert(i int, x T) {
	if i < 0 || i > v.length {
		panic("vector: index out of range")
	}
	var zero T
	v.Push(zero) // grow if needed; length++
	copy(v.data[i+1:v.length], v.data[i:v.length-1])
	v.data[i] = x
}

// Delete removes index i, shifting the tail left. Theta(n).
func (v *Vector[T]) Delete(i int) T {
	v.mustIndex(i)
	var zero T
	x := v.data[i]
	copy(v.data[i:v.length-1], v.data[i+1:v.length])
	v.length--
	v.data[v.length] = zero
	v.maybeShrink()
	return x
}

func (v *Vector[T]) maybeShrink() {
	c := len(v.data)
	if c > minCap && v.length*shrinkTrigger <= c {
		v.resize(c / shrinkFactor)
	}
}

// Slice returns a view of the live elements. The caller must not retain it
// across a mutation -- the backing array can be replaced by a resize.
func (v *Vector[T]) Slice() []T { return v.data[:v.length] }

// Reserve makes room for n more pushes without reallocating.
func (v *Vector[T]) Reserve(n int) {
	if need := v.length + n; need > len(v.data) {
		v.resize(need)
	}
}
```

**`vector_test.go`** — pass 2: prove it

```go
package vector

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// ============================================================================
// PASS 2 of the three-pass method: PROVE IT.
//
// Four layers, each catching what the others cannot:
//   1. a table          -- specific inputs, specific answers
//   2. invariants       -- what must be true after EVERY operation
//   3. a model oracle   -- agreement with an obviously-correct []T, on random
//                          operation sequences (this is the one that finds bugs)
//   4. allocation budget-- a Theta(1) claim, asserted rather than hoped for
// ============================================================================

// --- 1. the table ---------------------------------------------------------

func TestPushPop(t *testing.T) {
	var v Vector[int]

	if _, ok := v.Pop(); ok {
		t.Error("Pop on an empty vector reported ok=true")
	}
	if v.Len() != 0 {
		t.Errorf("zero vector Len = %d, want 0", v.Len())
	}

	for i := 1; i <= 5; i++ {
		v.Push(i * 10)
	}
	if got, want := v.Slice(), []int{10, 20, 30, 40, 50}; !slices.Equal(got, want) {
		t.Fatalf("after pushes = %v, want %v", got, want)
	}

	for i := 5; i >= 1; i-- {
		got, ok := v.Pop()
		if !ok || got != i*10 {
			t.Fatalf("Pop = (%d, %v), want (%d, true)", got, ok, i*10)
		}
	}
	if v.Len() != 0 {
		t.Errorf("after popping everything, Len = %d, want 0", v.Len())
	}
}

func TestInsertDelete(t *testing.T) {
	tests := []struct {
		name  string
		start []int
		op    func(*Vector[int])
		want  []int
	}{
		{"insert at front", []int{2, 3}, func(v *Vector[int]) { v.Insert(0, 1) }, []int{1, 2, 3}},
		{"insert in middle", []int{1, 3}, func(v *Vector[int]) { v.Insert(1, 2) }, []int{1, 2, 3}},
		{"insert at end", []int{1, 2}, func(v *Vector[int]) { v.Insert(2, 3) }, []int{1, 2, 3}},
		{"insert into empty", nil, func(v *Vector[int]) { v.Insert(0, 1) }, []int{1}},
		{"delete front", []int{1, 2, 3}, func(v *Vector[int]) { v.Delete(0) }, []int{2, 3}},
		{"delete middle", []int{1, 2, 3}, func(v *Vector[int]) { v.Delete(1) }, []int{1, 3}},
		{"delete last", []int{1, 2, 3}, func(v *Vector[int]) { v.Delete(2) }, []int{1, 2}},
		{"delete only", []int{1}, func(v *Vector[int]) { v.Delete(0) }, nil},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			var v Vector[int]
			for _, x := range tt.start {
				v.Push(x)
			}
			tt.op(&v)
			if got := v.Slice(); !slices.Equal(got, tt.want) {
				t.Errorf("got %v, want %v", got, tt.want)
			}
		})
	}
}

func TestPanicsOnBadIndex(t *testing.T) {
	for _, tt := range []struct {
		name string
		op   func(*Vector[int])
	}{
		{"At past end", func(v *Vector[int]) { v.At(3) }},
		{"At negative", func(v *Vector[int]) { v.At(-1) }},
		{"Delete past end", func(v *Vector[int]) { v.Delete(3) }},
		{"Insert past end+1", func(v *Vector[int]) { v.Insert(4, 0) }},
	} {
		t.Run(tt.name, func(t *testing.T) {
			var v Vector[int]
			v.Push(1)
			v.Push(2)
			v.Push(3)
			defer func() {
				if recover() == nil {
					t.Error("expected a panic, got none")
				}
			}()
			tt.op(&v)
		})
	}
}

// --- 2. the invariants ----------------------------------------------------

// checkInvariants asserts everything the type promises, at any moment.
func checkInvariants[T comparable](t *testing.T, v *Vector[T], where string) {
	t.Helper()
	if v.length < 0 || v.length > len(v.data) {
		t.Fatalf("%s: length %d out of range for cap %d", where, v.length, len(v.data))
	}
	var zero T
	for i := v.length; i < len(v.data); i++ {
		if v.data[i] != zero {
			t.Fatalf("%s: data[%d] past the length is %v, want the zero value "+
				"(a removed element is still reachable)", where, i, v.data[i])
		}
	}
}

func TestInvariantsHold(t *testing.T) {
	var v Vector[int]
	checkInvariants(t, &v, "zero value")

	for i := 0; i < 200; i++ {
		v.Push(i)
		checkInvariants(t, &v, fmt.Sprintf("after Push %d", i))
	}
	for i := 0; i < 100; i++ {
		v.Delete(0)
		checkInvariants(t, &v, fmt.Sprintf("after Delete %d", i))
	}
	for v.Len() > 0 {
		v.Pop()
		checkInvariants(t, &v, "after Pop")
	}
}

// --- 3. the model oracle --------------------------------------------------
//
// A plain []int is the reference implementation: obviously correct, and slow in
// ways nobody cares about here. Drive both with the same random operations and
// require they always agree.

func TestAgainstModel(t *testing.T) {
	rng := rand.New(rand.NewPCG(42, 7))

	const trials = 20_000
	var v Vector[int]
	var model []int

	for i := 0; i < trials; i++ {
		switch rng.IntN(5) {
		case 0, 1: // push (twice as likely, so it grows)
			x := rng.IntN(1000)
			v.Push(x)
			model = append(model, x)

		case 2: // pop
			got, ok := v.Pop()
			if len(model) == 0 {
				if ok {
					t.Fatalf("trial %d: Pop reported ok on an empty vector", i)
				}
				break
			}
			want := model[len(model)-1]
			model = model[:len(model)-1]
			if !ok || got != want {
				t.Fatalf("trial %d: Pop = (%d,%v), model says %d", i, got, ok, want)
			}

		case 3: // insert at a random position
			if len(model) == 0 {
				break
			}
			at := rng.IntN(len(model) + 1)
			x := rng.IntN(1000)
			v.Insert(at, x)
			model = slices.Insert(model, at, x)

		case 4: // delete a random position
			if len(model) == 0 {
				break
			}
			at := rng.IntN(len(model))
			got := v.Delete(at)
			want := model[at]
			model = slices.Delete(model, at, at+1)
			if got != want {
				t.Fatalf("trial %d: Delete(%d) = %d, model says %d", i, at, got, want)
			}
		}

		if v.Len() != len(model) {
			t.Fatalf("trial %d: Len = %d, model has %d", i, v.Len(), len(model))
		}
		if !slices.Equal(v.Slice(), model) {
			t.Fatalf("trial %d: contents diverged\n  vector %v\n  model  %v",
				i, v.Slice(), model)
		}
	}
	checkInvariants(t, &v, "after the whole run")
	t.Logf("%d random operations, final length %d, capacity %d", trials, v.Len(), v.Cap())
}

// --- 4. the allocation budget ---------------------------------------------

var sink int

func TestPushIsAmortizedConstant(t *testing.T) {
	// Reserve up front, then n pushes must allocate NOTHING.
	var v Vector[int]
	v.Reserve(1000)

	allocs := testing.AllocsPerRun(50, func() {
		v.length = 0 // reuse the same capacity each run
		for i := 0; i < 1000; i++ {
			v.Push(i)
		}
		sink = v.Len()
	})
	if allocs != 0 {
		t.Errorf("1000 pushes into reserved capacity allocated %.0f times, want 0", allocs)
	}
	t.Logf("1000 pushes after Reserve: %.0f allocations", allocs)
}

func TestGrowthIsGeometric(t *testing.T) {
	// The capacity must never grow by a constant amount: each resize should at
	// least double. That is the property the amortized bound depends on.
	var v Vector[int]
	prev := 0
	grows := 0
	for i := 0; i < 100_000; i++ {
		v.Push(i)
		if c := v.Cap(); c != prev {
			if prev > 0 && c < prev*2 {
				t.Fatalf("capacity grew %d -> %d, less than a doubling", prev, c)
			}
			prev = c
			grows++
		}
	}
	t.Logf("100,000 pushes caused %d resizes, final capacity %d", grows, v.Cap())
	if grows > 20 {
		t.Errorf("%d resizes for 100,000 pushes -- growth is not geometric", grows)
	}
}

// The shrink policy must not thrash (example 8).
func TestShrinkDoesNotThrash(t *testing.T) {
	var v Vector[int]
	for i := 0; i < 1025; i++ {
		v.Push(i)
	}
	v.Pop()

	capBefore := v.Cap()
	resizes := 0
	for i := 0; i < 10_000; i++ {
		v.Push(1)
		if v.Cap() != capBefore {
			resizes++
			capBefore = v.Cap()
		}
		v.Pop()
		if v.Cap() != capBefore {
			resizes++
			capBefore = v.Cap()
		}
	}
	if resizes != 0 {
		t.Errorf("push/pop at the capacity boundary caused %d resizes -- thrashing", resizes)
	}
	t.Logf("10,000 push/pop pairs at the boundary: %d resizes", resizes)
}

// And it must actually release memory when the vector really does empty out.
func TestShrinkReleasesMemory(t *testing.T) {
	var v Vector[int]
	for i := 0; i < 10_000; i++ {
		v.Push(i)
	}
	peak := v.Cap()
	for v.Len() > 0 {
		v.Pop()
	}
	if v.Cap() >= peak {
		t.Errorf("capacity stayed at %d after emptying (peak was %d)", v.Cap(), peak)
	}
	t.Logf("capacity %d at peak, %d after emptying", peak, v.Cap())
}
```

**`bench_test.go`** — pass 3: measure it

```go
package vector

import (
	"fmt"
	"testing"
)

// ============================================================================
// PASS 3: MEASURE IT.
//
// Two questions, and they are different:
//   1. does it have the complexity I claimed?      -> the size sweep
//   2. what does it cost against the alternative?  -> vs a native []T
// ============================================================================

// Always first. No number below this floor means anything (lesson 02).
func BenchmarkEmptyControl(b *testing.B) {
	for b.Loop() {
	}
}

// Push should be flat per element -- that is what amortized Theta(1) looks like.
func BenchmarkPush(b *testing.B) {
	for _, n := range []int{1_000, 10_000, 100_000} {
		b.Run(fmt.Sprintf("vector/n=%d", n), func(b *testing.B) {
			b.ReportAllocs()
			for b.Loop() {
				var v Vector[int]
				for i := 0; i < n; i++ {
					v.Push(i)
				}
			}
		})
		b.Run(fmt.Sprintf("native/n=%d", n), func(b *testing.B) {
			b.ReportAllocs()
			for b.Loop() {
				var xs []int
				for i := 0; i < n; i++ {
					xs = append(xs, i)
				}
			}
		})
		b.Run(fmt.Sprintf("reserved/n=%d", n), func(b *testing.B) {
			b.ReportAllocs()
			for b.Loop() {
				var v Vector[int]
				v.Reserve(n)
				for i := 0; i < n; i++ {
					v.Push(i)
				}
			}
		})
	}
}

// Insert at the front is Theta(n) per call, so this whole loop is Theta(n^2).
// The sweep should show the ~4x-per-doubling signature from lesson 03.
func BenchmarkInsertFront(b *testing.B) {
	for _, n := range []int{500, 1_000, 2_000, 4_000} {
		b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
			for b.Loop() {
				var v Vector[int]
				v.Reserve(n)
				for i := 0; i < n; i++ {
					v.Insert(0, i)
				}
			}
		})
	}
}

// Random access must not depend on n at all.
func BenchmarkAt(b *testing.B) {
	for _, n := range []int{1_000, 1_000_000} {
		var v Vector[int]
		v.Reserve(n)
		for i := 0; i < n; i++ {
			v.Push(i)
		}
		b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
			i := 0
			for b.Loop() {
				sink = v.At(i % n) // NOT i & 1023 -- that overruns n=1000
				i++
			}
		})
	}
}
```

**Run it:**

```bash
go test -v .
go test -bench=. -benchmem -run='^$' .
```

**Output:**

```
--- go test -v ---
--- PASS: TestPushPop (0.00s)
--- PASS: TestInsertDelete (0.00s)
    --- PASS: TestInsertDelete/insert_at_front (0.00s)
    --- PASS: TestInsertDelete/insert_in_middle (0.00s)
    --- PASS: TestInsertDelete/insert_at_end (0.00s)
    --- PASS: TestInsertDelete/insert_into_empty (0.00s)
    --- PASS: TestInsertDelete/delete_front (0.00s)
    --- PASS: TestInsertDelete/delete_middle (0.00s)
    --- PASS: TestInsertDelete/delete_last (0.00s)
    --- PASS: TestInsertDelete/delete_only (0.00s)
--- PASS: TestPanicsOnBadIndex (0.00s)
    --- PASS: TestPanicsOnBadIndex/At_past_end (0.00s)
    --- PASS: TestPanicsOnBadIndex/At_negative (0.00s)
    --- PASS: TestPanicsOnBadIndex/Delete_past_end (0.00s)
    --- PASS: TestPanicsOnBadIndex/Insert_past_end+1 (0.00s)
--- PASS: TestInvariantsHold (0.00s)
    vector_test.go:207: 20000 random operations, final length 4033, capacity 4096
--- PASS: TestAgainstModel (0.02s)
    vector_test.go:229: 1000 pushes after Reserve: 0 allocations
--- PASS: TestPushIsAmortizedConstant (0.00s)
    vector_test.go:248: 100,000 pushes caused 16 resizes, final capacity 131072
--- PASS: TestGrowthIsGeometric (0.00s)
    vector_test.go:279: 10,000 push/pop pairs at the boundary: 0 resizes
--- PASS: TestShrinkDoesNotThrash (0.00s)
    vector_test.go:295: capacity 16384 at peak, 4 after emptying
--- PASS: TestShrinkReleasesMemory (0.00s)
PASS
ok  	l06/ex16	0.164s

--- go test -bench=. -benchmem -run='^$' ---
goos: darwin
goarch: arm64
pkg: l06/ex16
cpu: Apple M4
BenchmarkEmptyControl-10    	677091841	         1.671 ns/op	       0 B/op	       0 allocs/op
BenchmarkPush/vector/n=1000-10         	  413733	      2916 ns/op	   16352 B/op	       9 allocs/op
BenchmarkPush/native/n=1000-10         	  404132	      3075 ns/op	   25208 B/op	      12 allocs/op
BenchmarkPush/reserved/n=1000-10       	  525883	      2248 ns/op	    8192 B/op	       1 allocs/op
BenchmarkPush/vector/n=10000-10        	   40626	     29511 ns/op	  262112 B/op	      13 allocs/op
BenchmarkPush/native/n=10000-10        	   35000	     34203 ns/op	  357625 B/op	      19 allocs/op
BenchmarkPush/reserved/n=10000-10      	   54010	     22139 ns/op	   81920 B/op	       1 allocs/op
BenchmarkPush/vector/n=100000-10       	    4458	    309454 ns/op	 2097131 B/op	      16 allocs/op
BenchmarkPush/native/n=100000-10       	    3250	    381914 ns/op	 4101393 B/op	      28 allocs/op
BenchmarkPush/reserved/n=100000-10     	    5353	    217233 ns/op	  802817 B/op	       1 allocs/op
BenchmarkInsertFront/n=500-10          	  116540	     10292 ns/op	    4096 B/op	       1 allocs/op
BenchmarkInsertFront/n=1000-10         	   32019	     37454 ns/op	    8192 B/op	       1 allocs/op
BenchmarkInsertFront/n=2000-10         	    8470	    144507 ns/op	   16384 B/op	       1 allocs/op
BenchmarkInsertFront/n=4000-10         	    2072	    579920 ns/op	   32768 B/op	       1 allocs/op
BenchmarkAt/n=1000-10                  	679784940	         1.761 ns/op	       0 B/op	       0 allocs/op
BenchmarkAt/n=1000000-10               	672972667	         1.800 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	l06/ex16	19.500s
```

Four things to take from the numbers:

- **`At` is 1.70 ns at n=1,000 and 1.72 ns at n=1,000,000.** Θ(1) confirmed — random access genuinely does not care how big the array is.
- **`InsertFront` grows ~4× per doubling** (10,485 → 37,397 → 140,416 → 579,820). That is Θ(n²) for the loop, exactly as lesson 03's ratio test predicts.
- **`Push` beats native `append`** at every size, and uses fewer allocations — because this `Vector` doubles strictly while Go's slices taper toward ×1.25 and therefore copy more (example 9). A hand-rolled structure is allowed to make a different trade.
- **`Reserve` then push is fastest of all, at exactly 1 allocation.** Same conclusion as every lesson so far: if you know n, say so.

The tests also caught a real bug while this was being written — not in the `Vector`, but in
`BenchmarkAt`, which indexed `i & 1023` into a 1,000-element vector. The bounds check turned a silent
out-of-range read into an immediate panic, which is precisely what it's for.

**Rust:**

```rust
use std::mem;

// =============================================================================
// vector.go -> the structure, with its invariant written down
// =============================================================================

// Vector<T> is the dynamic array from examples 7-9, finished: generic, with the
// hysteresis shrink policy, the slot hygiene, and a stated invariant.
//
// INVARIANT, true before and after every method:
//
//   length <= data.len()
//   data[length..] holds only T::default() values  (so no removed T is kept alive)
//
// The zero Vector<T> is a valid empty vector -- no constructor required. In Rust
// that is `Default`: `#[derive(Default)]` gives an empty boxed slice and length 0.
//
// Why Box<[T]> and not Vec<T>: a Vec already owns a growth policy, which is the
// thing being built (examples 7-9 made the same choice). `T: Default` stands in
// for Go's implicit zero value: every slot of a fresh array has to hold something.
#[derive(Default)]
struct Vector<T> {
    data: Box<[T]>,
    length: usize,
}

const MIN_CAP: usize = 4;
const GROWTH_FACTOR: usize = 2;
const SHRINK_TRIGGER: usize = 4; // shrink when length*4 <= cap ... (example 8)
const SHRINK_FACTOR: usize = 2; // ... and only halve, never shrink to fit

impl<T: Default> Vector<T> {
    fn len(&self) -> usize {
        self.length
    }

    fn cap(&self) -> usize {
        self.data.len()
    }

    /// Returns the element at i. Theta(1). Go's At returns a copy of T; Rust
    /// returns a borrow, so T need not be Copy.
    fn at(&self, i: usize) -> &T {
        self.must_index(i);
        &self.data[i]
    }

    /// Overwrites the element at i. Theta(1).
    fn set(&mut self, i: usize, x: T) {
        self.must_index(i);
        self.data[i] = x;
    }

    fn must_index(&self, i: usize) {
        // Go also checks `i < 0`. A usize cannot be negative, so that half of the
        // check -- and the "At negative" test case -- cannot exist in Rust.
        assert!(i < self.length, "vector: index out of range");
    }

    fn resize(&mut self, new_cap: usize) {
        let new_cap = new_cap.max(MIN_CAP);
        let mut bigger: Box<[T]> = (0..new_cap).map(|_| T::default()).collect();
        // Go copies the live prefix. Rust MOVES it (T need not be Clone): the
        // prefix is swapped into the new array, the old one is freed on assignment.
        bigger[..self.length].swap_with_slice(&mut self.data[..self.length]);
        self.data = bigger;
    }

    /// Appends x. Theta(1) amortized.
    fn push(&mut self, x: T) {
        if self.length == self.data.len() {
            self.resize((self.data.len() * GROWTH_FACTOR).max(MIN_CAP));
        }
        self.data[self.length] = x;
        self.length += 1;
    }

    /// Removes and returns the last element. Theta(1) amortized. Go returns
    /// `(T, bool)`; the Option is the same information in one type.
    fn pop(&mut self) -> Option<T> {
        if self.length == 0 {
            return None;
        }
        self.length -= 1;
        // take() moves the value out and leaves T::default() behind: the
        // invariant, nothing past length holds a live T.
        let x = mem::take(&mut self.data[self.length]);
        self.maybe_shrink();
        Some(x)
    }

    /// Places x at index i, shifting the tail right. Theta(n).
    fn insert(&mut self, i: usize, x: T) {
        assert!(i <= self.length, "vector: index out of range");
        self.push(T::default()); // grow if needed; length++
        // Go: copy(v.data[i+1:length], v.data[i:length-1]). Rust cannot hold two
        // overlapping &mut views, and `copy_within` needs T: Copy. Rotating the
        // tail right by one shifts it AND carries the fresh default to slot i.
        self.data[i..self.length].rotate_right(1);
        self.data[i] = x;
    }

    /// Removes index i, shifting the tail left. Theta(n).
    fn delete(&mut self, i: usize) -> T {
        self.must_index(i);
        // Rotating left shifts the tail and parks the deleted element in the last
        // live slot, from where take() moves it out and leaves the default behind.
        self.data[i..self.length].rotate_left(1);
        self.length -= 1;
        let x = mem::take(&mut self.data[self.length]);
        self.maybe_shrink();
        x
    }

    fn maybe_shrink(&mut self) {
        let c = self.data.len();
        if c > MIN_CAP && self.length * SHRINK_TRIGGER <= c {
            self.resize(c / SHRINK_FACTOR);
        }
    }

    /// A view of the live elements. Go's doc comment says "the caller must not
    /// retain it across a mutation -- the backing array can be replaced by a
    /// resize". In Rust that sentence is enforced: the slice borrows the vector,
    /// so `let s = v.as_slice(); v.push(1); s.len()` does not compile.
    fn as_slice(&self) -> &[T] {
        &self.data[..self.length]
    }

    /// Makes room for n more pushes without reallocating.
    fn reserve(&mut self, n: usize) {
        let need = self.length + n;
        if need > self.data.len() {
            self.resize(need);
        }
    }
}

// =============================================================================
// Shared test machinery: what `testing`, `testing.AllocsPerRun` and
// `math/rand/v2` give Go for free. Nothing here is about the Vector.
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
    // the numbers are visible with a plain `./vector`.
    pub fn log(line: &str) {
        let _ = writeln!(std::io::stdout(), "{line}");
    }

    /// A port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its IntN.
    /// With the same generator and seed (42, 7) the model oracle below sees the
    /// same 20,000 operations Go's test does.
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
        pub fn int_n(&mut self, n: u64) -> u64 {
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
}

// =============================================================================
// vector_test.go -> PASS 2 of the three-pass method: PROVE IT.
//
// Four layers, each catching what the others cannot:
//   1. a table          -- specific inputs, specific answers
//   2. invariants       -- what must be true after EVERY operation
//   3. a model oracle   -- agreement with an obviously-correct Vec<T>, on random
//                          operation sequences (this is the one that finds bugs)
//   4. allocation budget-- a Theta(1) claim, asserted rather than hoped for
// =============================================================================

#[cfg(test)]
mod tests {
    use super::*;
    use crate::harness::{Pcg, allocs_per_run, log};
    use std::fmt::Debug;
    use std::hint::black_box;
    use std::rc::Rc;

    fn filled(xs: &[i64]) -> Vector<i64> {
        let mut v = Vector::default();
        for &x in xs {
            v.push(x);
        }
        v
    }

    // --- 1. the table ---------------------------------------------------------

    #[test]
    fn push_pop() {
        let mut v: Vector<i64> = Vector::default();

        assert_eq!(v.pop(), None, "Pop on an empty vector returned a value");
        assert_eq!(v.len(), 0, "zero vector Len");

        for i in 1..=5 {
            v.push(i * 10);
        }
        assert_eq!(v.as_slice(), [10, 20, 30, 40, 50], "after pushes");

        for i in (1..=5).rev() {
            assert_eq!(v.pop(), Some(i * 10), "Pop");
        }
        assert_eq!(v.len(), 0, "after popping everything, Len");
    }

    // libtest has no subtests (Go's t.Run). A macro turns each row of the table
    // into its own #[test], so every case still gets its own line in the output
    // and can be run alone: `./vector insert_delete::delete_middle`.
    mod insert_delete {
        use super::*;

        macro_rules! cases {
            ($($name:ident: $start:expr, $op:expr => $want:expr;)*) => {$(
                #[test]
                fn $name() {
                    let mut v = filled(&$start);
                    let op: fn(&mut Vector<i64>) = $op;
                    op(&mut v);
                    let want: &[i64] = &$want;
                    assert_eq!(v.as_slice(), want);
                }
            )*};
        }

        cases! {
            insert_at_front: [2, 3], |v| v.insert(0, 1) => [1, 2, 3];
            insert_in_middle: [1, 3], |v| v.insert(1, 2) => [1, 2, 3];
            insert_at_end: [1, 2], |v| v.insert(2, 3) => [1, 2, 3];
            insert_into_empty: [], |v| v.insert(0, 1) => [1];
            delete_front: [1, 2, 3], |v| { v.delete(0); } => [2, 3];
            delete_middle: [1, 2, 3], |v| { v.delete(1); } => [1, 3];
            delete_last: [1, 2, 3], |v| { v.delete(2); } => [1, 2];
            delete_only: [1], |v| { v.delete(0); } => [];
        }
    }

    // Go's recover() in a deferred func becomes #[should_panic(expected = ..)].
    // The Go table has a fourth row, "At negative"; with usize indexes it does
    // not compile, which is the test.
    mod panics_on_bad_index {
        use super::*;

        macro_rules! cases {
            ($($name:ident: $op:expr;)*) => {$(
                #[test]
                #[should_panic(expected = "vector: index out of range")]
                fn $name() {
                    let mut v = filled(&[1, 2, 3]);
                    let op: fn(&mut Vector<i64>) = $op;
                    op(&mut v);
                }
            )*};
        }

        cases! {
            at_past_end: |v| { v.at(3); };
            delete_past_end: |v| { v.delete(3); };
            insert_past_end_plus_1: |v| v.insert(4, 0);
        }
    }

    // The Go suite never calls Set, and clippy -D warnings rejects a method no
    // test or caller uses (as would a reviewer): one small test closes the gap.
    #[test]
    fn set_overwrites() {
        let mut v = filled(&[1, 2, 3]);
        v.set(1, 20);
        assert_eq!(v.as_slice(), [1, 20, 3]);
    }

    // --- 2. the invariants ----------------------------------------------------

    /// check_invariants asserts everything the type promises, at any moment.
    fn check_invariants<T: Default + PartialEq + Debug>(v: &Vector<T>, place: &str) {
        assert!(
            v.length <= v.data.len(),
            "{place}: length {} out of range for cap {}",
            v.length,
            v.data.len()
        );
        let zero = T::default();
        for i in v.length..v.data.len() {
            assert!(
                v.data[i] == zero,
                "{place}: data[{i}] past the length is {:?}, want the zero value \
                 (a removed element is still reachable)",
                v.data[i]
            );
        }
    }

    #[test]
    fn invariants_hold() {
        let mut v: Vector<i64> = Vector::default();
        check_invariants(&v, "zero value");

        for i in 0..200 {
            v.push(i);
            check_invariants(&v, &format!("after Push {i}"));
        }
        for i in 0..100 {
            v.delete(0);
            check_invariants(&v, &format!("after Delete {i}"));
        }
        while v.len() > 0 {
            v.pop();
            check_invariants(&v, "after Pop");
        }
    }

    // Rust addition. The invariant exists so that a removed T is not kept alive;
    // Go's test checks that with ints (zero values), which cannot show a leak. An
    // Rc<()> can: its strong count says exactly how many owners are still alive.
    #[test]
    fn removed_elements_are_released() {
        let token = Rc::new(());
        let mut v: Vector<Rc<()>> = Vector::default();
        for _ in 0..8 {
            v.push(Rc::clone(&token));
        }
        assert_eq!(Rc::strong_count(&token), 1 + 8);

        drop(v.pop());
        drop(v.delete(0));
        // 6 live elements + the token itself: nothing is parked in a spare slot
        assert_eq!(Rc::strong_count(&token), 1 + 6);

        while v.pop().is_some() {}
        assert_eq!(Rc::strong_count(&token), 1);
    }

    // --- 3. the model oracle --------------------------------------------------
    //
    // A plain Vec<i64> is the reference implementation: obviously correct, and
    // slow in ways nobody cares about here. Drive both with the same random
    // operations and require they always agree.

    #[test]
    fn against_model() {
        let mut rng = Pcg::new(42, 7);

        const TRIALS: usize = 20_000;
        let mut v: Vector<i64> = Vector::default();
        let mut model: Vec<i64> = Vec::new();

        for i in 0..TRIALS {
            match rng.int_n(5) {
                0 | 1 => {
                    // push (twice as likely, so it grows)
                    let x = rng.int_n(1000) as i64;
                    v.push(x);
                    model.push(x);
                }
                2 => {
                    // pop: one assert covers "ok on an empty vector", the value,
                    // and the missing value, because Option<T> carries all three
                    assert_eq!(v.pop(), model.pop(), "trial {i}: Pop");
                }
                3 if !model.is_empty() => {
                    // insert at a random position
                    let at = rng.int_n(model.len() as u64 + 1) as usize;
                    let x = rng.int_n(1000) as i64;
                    v.insert(at, x);
                    model.insert(at, x);
                }
                4 if !model.is_empty() => {
                    // delete a random position
                    let at = rng.int_n(model.len() as u64) as usize;
                    let got = v.delete(at);
                    let want = model.remove(at);
                    assert_eq!(got, want, "trial {i}: Delete({at})");
                }
                _ => {} // insert/delete on an empty model: nothing to do
            }

            assert_eq!(v.len(), model.len(), "trial {i}: Len");
            assert_eq!(v.as_slice(), model, "trial {i}: contents diverged");
        }
        check_invariants(&v, "after the whole run");
        log(&format!(
            "{TRIALS} random operations, final length {}, capacity {}",
            v.len(),
            v.cap()
        ));
    }

    // --- 4. the allocation budget ---------------------------------------------

    #[test]
    fn push_is_amortized_constant() {
        // Reserve up front, then n pushes must allocate NOTHING.
        let mut v: Vector<i64> = Vector::default();
        v.reserve(1000);

        let allocs = allocs_per_run(50, || {
            v.length = 0; // reuse the same capacity each run (a test-only hack)
            for i in 0..1000 {
                v.push(i);
            }
            black_box(v.len());
        });
        assert_eq!(
            allocs, 0,
            "1000 pushes into reserved capacity allocated {allocs} times, want 0"
        );
        log(&format!("1000 pushes after Reserve: {allocs} allocations"));
    }

    #[test]
    fn growth_is_geometric() {
        // The capacity must never grow by a constant amount: each resize should at
        // least double. That is the property the amortized bound depends on.
        let mut v: Vector<i64> = Vector::default();
        let mut prev = 0;
        let mut grows = 0;
        for i in 0..100_000 {
            v.push(i);
            let c = v.cap();
            if c != prev {
                assert!(
                    prev == 0 || c >= prev * 2,
                    "capacity grew {prev} -> {c}, less than a doubling"
                );
                prev = c;
                grows += 1;
            }
        }
        log(&format!(
            "100,000 pushes caused {grows} resizes, final capacity {}",
            v.cap()
        ));
        assert!(
            grows <= 20,
            "{grows} resizes for 100,000 pushes -- growth is not geometric"
        );
    }

    // The shrink policy must not thrash (example 8).
    #[test]
    fn shrink_does_not_thrash() {
        let mut v: Vector<i64> = Vector::default();
        for i in 0..1025 {
            v.push(i);
        }
        v.pop();

        let mut cap_before = v.cap();
        let mut resizes = 0;
        for _ in 0..10_000 {
            v.push(1);
            if v.cap() != cap_before {
                resizes += 1;
                cap_before = v.cap();
            }
            v.pop();
            if v.cap() != cap_before {
                resizes += 1;
                cap_before = v.cap();
            }
        }
        assert_eq!(
            resizes, 0,
            "push/pop at the capacity boundary caused {resizes} resizes -- thrashing"
        );
        log(&format!(
            "10,000 push/pop pairs at the boundary: {resizes} resizes"
        ));
    }

    // And it must actually release memory when the vector really does empty out.
    #[test]
    fn shrink_releases_memory() {
        let mut v: Vector<i64> = Vector::default();
        for i in 0..10_000 {
            v.push(i);
        }
        let peak = v.cap();
        while v.pop().is_some() {}
        assert!(
            v.cap() < peak,
            "capacity stayed at {} after emptying (peak was {peak})",
            v.cap()
        );
        log(&format!(
            "capacity {peak} at peak, {} after emptying",
            v.cap()
        ));
    }
}

// =============================================================================
// bench_test.go -> PASS 3: MEASURE IT.
//
// Two questions, and they are different:
//   1. does it have the complexity I claimed?      -> the size sweep
//   2. what does it cost against the alternative?  -> vs a native Vec<T>
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
    use crate::harness::{allocs_of, log};
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
    fn run(name: &str, mut f: impl FnMut()) -> f64 {
        let (allocs, bytes) = allocs_of(&mut f);
        let ns = ns_per_op(&mut f);
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

        // Push should be flat per element -- that is what amortized Theta(1)
        // looks like. black_box(n) hides the constant from the optimizer.
        for n in [1_000, 10_000, 100_000] {
            let n = black_box(n);
            run(&format!("Push/vector/n={n}"), || {
                let mut v: Vector<i64> = Vector::default();
                for i in 0..n {
                    v.push(i as i64);
                }
                black_box(&v);
            });
            run(&format!("Push/native/n={n}"), || {
                let mut xs: Vec<i64> = Vec::new();
                for i in 0..n {
                    xs.push(i as i64);
                }
                black_box(&xs);
            });
            run(&format!("Push/reserved/n={n}"), || {
                let mut v: Vector<i64> = Vector::default();
                v.reserve(n);
                for i in 0..n {
                    v.push(i as i64);
                }
                black_box(&v);
            });
            // Rust addition: std's answer to "if you know n, say so".
            run(&format!("Push/native-reserved/n={n}"), || {
                let mut xs: Vec<i64> = Vec::with_capacity(n);
                for i in 0..n {
                    xs.push(i as i64);
                }
                black_box(&xs);
            });
        }

        // Insert at the front is Theta(n) per call, so this whole loop is Theta(n^2).
        // The sweep should show the ~4x-per-doubling signature from lesson 03.
        let mut times = Vec::new();
        for n in [500, 1_000, 2_000, 4_000] {
            let n = black_box(n);
            times.push(run(&format!("InsertFront/n={n}"), || {
                let mut v: Vector<i64> = Vector::default();
                v.reserve(n);
                for i in 0..n {
                    v.insert(0, i as i64);
                }
                black_box(&v);
            }));
        }
        let ratios: Vec<String> = times
            .windows(2)
            .map(|w| format!("{:.2}", w[1] / w[0]))
            .collect();
        log(&format!(
            "  InsertFront ratio per doubling: {}",
            ratios.join(", ")
        ));

        // Rust addition. std's Vec grows through realloc, which may extend the
        // block where it sits; Vector::resize always allocates a new array and
        // moves the prefix. Count how often Vec's buffer actually moved.
        let n = black_box(100_000);
        let mut xs: Vec<i64> = Vec::new();
        let (mut grows, mut moved) = (0, 0);
        let (mut cap, mut ptr) = (xs.capacity(), xs.as_ptr());
        for i in 0..n {
            xs.push(i as i64);
            if xs.capacity() != cap {
                grows += 1;
                moved += usize::from(xs.as_ptr() != ptr);
                (cap, ptr) = (xs.capacity(), xs.as_ptr());
            }
        }
        log(&format!(
            "  std Vec, {n} pushes: {grows} growths (the first is the initial allocation), \
             the buffer moved in {moved}"
        ));

        // Random access must not depend on n at all.
        for n in [1_000, 1_000_000] {
            let n = black_box(n);
            let mut v: Vector<i64> = Vector::default();
            v.reserve(n);
            for i in 0..n {
                v.push(i as i64);
            }
            let mut i = 0;
            run(&format!("At/n={n}"), || {
                // NOT i & 1023 -- that overruns n=1000
                black_box(*v.at(i % n));
                i += 1;
            });
        }
    }
}
```

**Run it (Rust):**

```bash
rustc --edition 2024 -O --test vector.rs -o vector && ./vector
```

**Rust sample output:**

```

running 21 tests
test tests::insert_delete::delete_front ... ok
test tests::insert_delete::delete_middle ... ok
test tests::insert_delete::delete_last ... ok
test tests::insert_delete::delete_only ... ok
test tests::insert_delete::insert_at_end ... ok
test tests::insert_delete::insert_at_front ... ok
test tests::insert_delete::insert_in_middle ... ok
100,000 pushes caused 16 resizes, final capacity 131072
1000 pushes after Reserve: 0 allocations
test tests::growth_is_geometric ... ok
test tests::insert_delete::insert_into_empty ... ok
test tests::invariants_hold ... ok
test tests::panics_on_bad_index::at_past_end - should panic ... ok
test tests::panics_on_bad_index::delete_past_end - should panic ... ok
test tests::panics_on_bad_index::insert_past_end_plus_1 - should panic ... ok
test tests::push_is_amortized_constant ... ok
test tests::push_pop ... ok
test tests::removed_elements_are_released ... ok
test tests::set_overwrites ... ok
10,000 push/pop pairs at the boundary: 0 resizes
test tests::shrink_does_not_thrash ... ok
capacity 16384 at peak, 4 after emptying
test tests::shrink_releases_memory ... ok
20000 random operations, final length 4033, capacity 4096
test tests::against_model ... ok
EmptyControl                          0.3 ns/op          0 B/op    0 allocs/op
Push/vector/n=1000                 1151.2 ns/op      16352 B/op    9 allocs/op
Push/native/n=1000                  653.7 ns/op      16352 B/op    9 allocs/op
Push/reserved/n=1000                593.1 ns/op       8000 B/op    1 allocs/op
Push/native-reserved/n=1000         367.8 ns/op       8000 B/op    1 allocs/op
Push/vector/n=10000               13515.8 ns/op     262112 B/op   13 allocs/op
Push/native/n=10000                6412.2 ns/op     262112 B/op   13 allocs/op
Push/reserved/n=10000              6067.6 ns/op      80000 B/op    1 allocs/op
Push/native-reserved/n=10000       3407.6 ns/op      80000 B/op    1 allocs/op
Push/vector/n=100000             203562.2 ns/op    2097120 B/op   16 allocs/op
Push/native/n=100000              62097.9 ns/op    2097120 B/op   16 allocs/op
Push/reserved/n=100000            55636.6 ns/op     800000 B/op    1 allocs/op
Push/native-reserved/n=100000      34223.0 ns/op     800000 B/op    1 allocs/op
InsertFront/n=500                 10498.4 ns/op       4000 B/op    1 allocs/op
InsertFront/n=1000                40000.0 ns/op       8000 B/op    1 allocs/op
InsertFront/n=2000               158358.1 ns/op      16000 B/op    1 allocs/op
InsertFront/n=4000               631058.2 ns/op      32000 B/op    1 allocs/op
  InsertFront ratio per doubling: 3.81, 3.96, 3.99
  std Vec, 100000 pushes: 16 growths (the first is the initial allocation), the buffer moved in 5
At/n=1000                             0.9 ns/op          0 B/op    0 allocs/op
At/n=1000000                          0.7 ns/op          0 B/op    0 allocs/op
test bench::benchmarks ... ok

test result: ok. 21 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 5.53s
```

> **Go vs Rust:** The five logged lines are identical to Go's `t.Logf` lines (20000 operations ending at length 4033 / capacity 4096, 16 resizes to 131072, 0 resizes on the boundary, 16384 -> 4) because a port of Go's PCG feeds the model oracle the same 20,000 operations. `Vector`'s rows match Go's too (9 / 13 / 16 allocations, 16352 / 262112 B at n=1,000 / 10,000), and `reserved` is 1 allocation of exactly 8000 B where Go shows 8192 (size-class rounding). The source differs where the languages do: `Pop` returns `Option`, the "At negative" row cannot be written because `usize` is never negative, `as_slice` cannot be kept across a `push` (a borrow-checker error, not a doc comment), and libtest has no subtests, so a macro makes each table row its own test. Two Rust-only additions: `set_overwrites` (the Go suite never calls `Set`) and `removed_elements_are_released`, which uses `Rc` counts to show what the zero-slot invariant is for.
> The Go prose's claim "Push beats native append at every size, and uses fewer allocations" does not hold here. Over 8 separate runs (this block is another one) std's `Vec::push` beat `Vector` in all 24 comparisons: 1.1-1.8x at n=1,000, 1.8-2.4x at 10,000, 2.7-3.6x at 100,000, with the same 9 / 13 / 16 allocation events and bytes, since std's `Vec` also doubles. One cause was measured (the last log line): across the 16 growths to 100,000 elements the `Vec` buffer moved in only 4-6 (the first is the initial allocation), so most growths extended in place and copied nothing, while `Vector::resize` allocates, zero-fills and moves every time. That is not all of it: with growth removed, `reserved` was still 1.2-2.6x slower than std's `with_capacity` (an added row) in all 24, plausibly the zero-fill and the per-push bounds check, not measured. `reserved` beat the growing `Vec` in 20 of 24 comparisons, but `with_capacity` was fastest in all 24. The other two Go readings hold: `At` is flat at 0.6-1.0 ns for n=1,000 and n=1,000,000, and `InsertFront` grows 3.4-4.3x per doubling (24 ratios).

**Complexity:** `At`/`Set`/`Push`/`Pop` Θ(1) (amortized for the last two) · `Insert`/`Delete` Θ(n) · Θ(n) space with capacity between 1× and 2× the length

---

> That's Part 2's foundation. Next: [07 — Linked Lists](../../07-linked-lists.md) — the mirror image
> of this lesson's trade-off, and an honest look at how rarely it wins.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
