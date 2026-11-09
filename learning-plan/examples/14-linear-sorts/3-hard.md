# Step 14 — Linear-Time Sorts · 🔴 Hard

Examples **12–16**: where radix and counting sort start to beat `slices.Sort`, timestamps and IDs
with passes to skip, strings with long shared prefixes, American flag sort in place, and the capstone.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic — your output should match character for character;
**Sample output:** ones print timings, which will differ on your machine.

**Run the Rust twin of any example** (std only — nothing to install but `rustc`):

```bash
mkdir -p /tmp/dsa-rs && cd /tmp/dsa-rs
# paste the Rust block into main.rs, then:
rustc --edition 2024 -O main.rs && ./main
```

Every example below shows the Go program and its output first, then its **Rust twin** with its own real
output and a short **Go vs Rust** note. Always build with `-O` — the examples measure time. Where a
twin needs a different command (`--test`, a flag), it is shown under **Run it (Rust):** beneath the code.

> ← Back to [🟡 medium](2-medium.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. The crossover against `slices.Sort`

`🔴 hard` · *Measured crossover*

Linear does not mean fast. LSD radix sort on 64-bit keys with 8-bit digits is Θ(8n), but every one of
its 8 passes reads n keys, scatters them over 256 destinations and writes into a second n-element
buffer, and every call clears 8 × 256 counters before it has looked at a key. pdqsort does about
n·log₂ n comparisons, but on a contiguous `[]uint64` those are cheap. So for each n there is a
winner, and the only way to know which is to measure it: this program times LSD with 8-bit and
16-bit digits against `slices.Sort` on random keys, and counting sort against `slices.Sort` on keys
below 1024, and computes the crossover from its own table.

**Steps:**

1. Check that every sort agrees with `slices.Sort` at every size before starting the clock.
2. Time `copy pristine → work, then sort(work)` so that each iteration really sorts unsorted data,
   time the same loop with the copy only, and subtract that.
3. Read the verdict the program computes from its table, and the allocation counts that explain part
   of the small-n cost.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/rand/v2"
	"runtime/debug"
	"slices"
	"testing"
	"time"
)

// "Linear" is a statement about growth, not about speed. LSD radix sort on
// 64-bit keys with 8-bit digits makes 8 passes; each pass reads n keys,
// scatters them to 256 places and needs a second buffer of n. pdqsort
// (slices.Sort) does ~n*log2(n) comparisons, but on a contiguous []uint64
// they are cheap and cache-friendly. So there is a crossover, and the only
// honest way to find it is to measure.

// lsd8 sorts 64-bit keys with eight 8-bit counting-sort passes.
// One allocation per call: the ping-pong buffer.
func lsd8(xs []uint64) {
	buf := make([]uint64, len(xs))
	src, dst := xs, buf
	for shift := uint(0); shift < 64; shift += 8 {
		var count [256]int // 2 KiB: lives on the stack
		for _, x := range src {
			count[(x>>shift)&0xFF]++
		}
		pos := 0
		for d, c := range count { // counts -> start positions
			count[d] = pos
			pos += c
		}
		for _, x := range src { // stable scatter
			d := (x >> shift) & 0xFF
			dst[count[d]] = x
			count[d]++
		}
		src, dst = dst, src // 8 passes: an even number, so we end in xs
	}
}

// lsd16 is the same with 16-bit digits: 4 passes, but 65,536 counters.
func lsd16(xs []uint64) {
	buf := make([]uint64, len(xs))
	src, dst := xs, buf
	for shift := uint(0); shift < 64; shift += 16 {
		// 512 KiB: above the compiler's 128 KiB MaxStackVarSize, so Go puts
		// this array on the heap -- one more allocation on every pass.
		var count [1 << 16]int
		for _, x := range src {
			count[(x>>shift)&0xFFFF]++
		}
		pos := 0
		for d, c := range count {
			count[d] = pos
			pos += c
		}
		for _, x := range src {
			d := (x >> shift) & 0xFFFF
			dst[count[d]] = x
			count[d]++
		}
		src, dst = dst, src
	}
}

// counting1024 sorts bare keys < 1024 by rewriting them from their counts.
func counting1024(xs []uint64) {
	var count [1024]int
	for _, x := range xs {
		count[x]++
	}
	i := 0
	for v, c := range count {
		for ; c > 0; c-- {
			xs[i] = uint64(v)
			i++
		}
	}
}

func stdSort(xs []uint64) { slices.Sort(xs) }
func copyOnly([]uint64)   {} // the control: same loop, same call, no sort

// nsPerCall times `copy pristine -> work, then f(work)`. The copy is INSIDE
// the timed loop on purpose: sorting an already-sorted slice measures
// nothing. The copy's own cost is measured separately (copyOnly) and
// subtracted by the caller. Best of 3 batches of >= 20 ms each.
func nsPerCall(work, pristine []uint64, f func([]uint64)) float64 {
	batch := func(reps int) time.Duration {
		start := time.Now()
		for range reps {
			copy(work, pristine)
			f(work)
		}
		return time.Since(start)
	}
	reps := 1
	for batch(reps) < 20*time.Millisecond {
		reps *= 2
	}
	best := math.Inf(1)
	for range 3 {
		best = min(best, float64(batch(reps).Nanoseconds())/float64(reps))
	}
	return best
}

func label(n int) string {
	switch {
	case n >= 1<<20:
		return fmt.Sprintf("%dM", n>>20)
	case n >= 1<<10:
		return fmt.Sprintf("%dK", n>>10)
	}
	return fmt.Sprint(n)
}

// crossover: the first n where `fast` beats `base`, and the smallest n from
// which it wins at EVERY larger n in the table (-1 = never).
func crossover(ns []int, fast, base []float64) (first, from int) {
	first, from = -1, -1
	for i := range ns {
		if fast[i] < base[i] && first < 0 {
			first = ns[i]
		}
	}
	for i := len(ns) - 1; i >= 0 && fast[i] < base[i]; i-- {
		from = ns[i]
	}
	return
}

func verdict(name, opp string, ns []int, fast, base []float64) {
	first, from := crossover(ns, fast, base)
	last := len(ns) - 1
	switch {
	case first < 0:
		fmt.Printf("  %-12s never beats %s in this range (at n=%s: %.2fx its time)\n",
			name, opp, label(ns[last]), fast[last]/base[last])
	case from < 0:
		fmt.Printf("  %-12s first beats %s at n=%s, but loses again at n=%s\n",
			name, opp, label(first), label(ns[last]))
	case first == from:
		fmt.Printf("  %-12s beats %s from n=%s on (at n=%s: %.1fx faster)\n",
			name, opp, label(from), label(ns[last]), base[last]/fast[last])
	default:
		fmt.Printf("  %-12s first beats %s at n=%s, and at every n from %s on (at n=%s: %.1fx faster)\n",
			name, opp, label(first), label(from), label(ns[last]), base[last]/fast[last])
	}
}

func main() {
	// Each LSD call allocates a fresh buffer of n; at n = 4M that is 32 MB of
	// garbage per call. A soft memory limit makes the GC collect it sooner, so
	// the program stays near 200 MB instead of letting the heap double.
	debug.SetMemoryLimit(200 << 20)

	ns := []int{16, 64, 256, 1 << 10, 4 << 10, 16 << 10, 64 << 10, 256 << 10, 1 << 20, 4 << 20}
	maxN := ns[len(ns)-1]

	rng := rand.New(rand.NewPCG(1, 2))
	wide := make([]uint64, maxN)  // random 64-bit keys
	small := make([]uint64, maxN) // random keys < 1024
	for i := range wide {
		wide[i] = rng.Uint64()
	}
	for i := range small {
		small[i] = rng.Uint64() % 1024
	}
	work := make([]uint64, maxN)
	ref := make([]uint64, maxN)

	// 1. Correctness before any clock is started.
	ok := true
	for _, n := range ns {
		for _, c := range []struct {
			in []uint64
			f  func([]uint64)
		}{{wide, lsd8}, {wide, lsd16}, {small, lsd8}, {small, counting1024}} {
			r, w := ref[:n], work[:n]
			copy(r, c.in[:n])
			slices.Sort(r)
			copy(w, c.in[:n])
			c.f(w)
			ok = ok && slices.Equal(r, w)
		}
	}
	fmt.Printf("1. lsd8, lsd16 (random keys) and lsd8, counting (keys < 1024) equal slices.Sort at all %d sizes: %v\n\n", len(ns), ok)

	// 2. The clock. ns per ELEMENT, copy cost subtracted.
	fmt.Println("2. ns/element (copy-only control subtracted; best of 3)")
	fmt.Printf("%6s %-30s | %-21s |\n", "", "random uint64 keys", "keys < 1024")
	fmt.Printf("%6s %11s %8s %9s | %11s %9s | %5s\n", "n", "slices.Sort", "lsd8", "lsd16", "slices.Sort", "counting", "copy")
	var stdW, l8, l16, stdS, cnt []float64
	for _, n := range ns {
		w := work[:n]
		ctrl := nsPerCall(w, wide[:n], copyOnly)
		per := func(in []uint64, f func([]uint64)) float64 {
			return (nsPerCall(w, in[:n], f) - ctrl) / float64(n)
		}
		stdW = append(stdW, per(wide, stdSort))
		l8 = append(l8, per(wide, lsd8))
		l16 = append(l16, per(wide, lsd16))
		stdS = append(stdS, per(small, stdSort))
		cnt = append(cnt, per(small, counting1024))
		i := len(stdW) - 1
		fmt.Printf("%6s %11.2f %8.2f %9.2f | %11.2f %9.2f | %5.2f\n",
			label(n), stdW[i], l8[i], l16[i], stdS[i], cnt[i], ctrl/float64(n))
	}

	// 3. The verdict, computed from the table above.
	fmt.Println("\n3. Crossover (from this run's numbers)")
	verdict("lsd8", "slices.Sort", ns, l8, stdW)
	verdict("lsd16", "slices.Sort", ns, l16, stdW)
	verdict("counting", "slices.Sort", ns, cnt, stdS)
	fmt.Printf("  fixed cost per call: lsd8 clears %d counters, lsd16 %d, counting %d\n",
		8*256, 4*(1<<16), 1024)

	// 4. Allocations per call (deterministic).
	fmt.Println("\n4. Heap allocations per call (testing.AllocsPerRun)")
	fmt.Printf("%-24s %8s %8s\n", "", "n=16", "n=1M")
	for _, c := range []struct {
		name string
		f    func([]uint64)
	}{
		{"slices.Sort", stdSort},
		{"slices.SortStableFunc", func(xs []uint64) { slices.SortStableFunc(xs, cmp.Compare[uint64]) }},
		{"lsd8", lsd8},
		{"lsd16", lsd16},
		{"counting (keys < 1024)", counting1024},
	} {
		row := fmt.Sprintf("%-24s", c.name)
		for _, n := range []int{16, 1 << 20} {
			in := wide
			if c.name == "counting (keys < 1024)" {
				in = small
			}
			w := work[:n]
			a := testing.AllocsPerRun(3, func() { copy(w, in[:n]); c.f(w) })
			row += fmt.Sprintf(" %8.0f", a)
		}
		fmt.Println(row)
	}
}
```

**Sample output:**

```
1. lsd8, lsd16 (random keys) and lsd8, counting (keys < 1024) equal slices.Sort at all 10 sizes: true

2. ns/element (copy-only control subtracted; best of 3)
       random uint64 keys             | keys < 1024           |
     n slices.Sort     lsd8     lsd16 | slices.Sort  counting |  copy
    16        2.30    72.25   8239.15 |        2.58     37.27 |  0.13
    64        3.66    22.70   2013.74 |        3.80      9.27 |  0.09
   256        6.13    10.10    528.30 |        6.64      3.47 |  0.07
    1K        8.72     6.95    141.86 |        8.76      1.60 |  0.07
    4K       10.32     6.21     47.50 |       10.19      0.81 |  0.11
   16K       29.04     6.65     15.75 |       21.41      0.68 |  0.12
   64K       44.25     6.43     10.43 |       26.17      0.63 |  0.12
  256K       51.73     6.41      8.61 |       27.85      0.61 |  0.11
    1M       57.93     7.58      8.35 |       28.51      0.58 |  0.11
    4M       63.56     7.69      9.35 |       28.79      0.57 |  0.18

3. Crossover (from this run's numbers)
  lsd8         beats slices.Sort from n=1K on (at n=4M: 8.3x faster)
  lsd16        beats slices.Sort from n=16K on (at n=4M: 6.8x faster)
  counting     beats slices.Sort from n=256 on (at n=4M: 50.8x faster)
  fixed cost per call: lsd8 clears 2048 counters, lsd16 262144, counting 1024

4. Heap allocations per call (testing.AllocsPerRun)
                             n=16     n=1M
slices.Sort                     0        0
slices.SortStableFunc           0        0
lsd8                            1        1
lsd16                           5        5
counting (keys < 1024)          0        0
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// "Linear" is a statement about growth, not about speed. LSD radix sort on
// 64-bit keys with 8-bit digits makes 8 passes; each pass reads n keys,
// scatters them to 256 places and needs a second buffer of n. ipnsort
// (sort_unstable, Rust's counterpart of slices.Sort) does ~n*log2(n)
// comparisons, but on a contiguous [u64] they are cheap. So there is a
// crossover, and the only honest way to find it is to measure.

// ---- allocation counter: Rust's stand-in for testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static TOTAL: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn alloc_zeroed(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc_zeroed(l) }
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
fn allocs_of(f: impl FnOnce()) -> (usize, usize) {
    let (a0, t0) = (ALLOCS.load(Relaxed), TOTAL.load(Relaxed));
    f();
    (ALLOCS.load(Relaxed) - a0, TOTAL.load(Relaxed) - t0)
}

// ---- Go's math/rand/v2 PCG (PCG-DXSM), so the keys are Go's keys ----
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
}

/// Every sort here, as a plain function pointer (Go: a `func([]uint64)` value).
type Sorter = fn(&mut [u64]);

// lsd8 sorts 64-bit keys with eight 8-bit counting-sort passes.
// One allocation per call: the ping-pong buffer.
fn lsd8(xs: &mut [u64]) {
    let mut buf = vec![0u64; xs.len()];
    // Two &mut slices we can swap: Go swaps slice headers, Rust swaps the
    // borrows. Both point at disjoint memory, so the borrow checker is happy.
    let (mut src, mut dst): (&mut [u64], &mut [u64]) = (xs, &mut buf);
    for shift in (0..64).step_by(8) {
        let mut count = [0usize; 256]; // 2 KiB on the stack
        for &x in src.iter() {
            count[((x >> shift) & 0xFF) as usize] += 1;
        }
        let mut pos = 0;
        for c in count.iter_mut() {
            // counts -> start positions
            let k = *c;
            *c = pos;
            pos += k;
        }
        for &x in src.iter() {
            // stable scatter
            let d = ((x >> shift) & 0xFF) as usize;
            dst[count[d]] = x;
            count[d] += 1;
        }
        std::mem::swap(&mut src, &mut dst); // 8 passes: we end in xs
    }
}

// lsd16 is the same with 16-bit digits: 4 passes, but 65,536 counters.
fn lsd16(xs: &mut [u64]) {
    let mut buf = vec![0u64; xs.len()];
    let (mut src, mut dst): (&mut [u64], &mut [u64]) = (xs, &mut buf);
    for shift in (0..64).step_by(16) {
        // 512 KiB, and Rust keeps it on the stack: an array is a local value,
        // there is no size threshold that moves it to the heap (the main
        // thread's 8 MB stack is the only limit). So: no allocation here.
        let mut count = [0usize; 1 << 16];
        for &x in src.iter() {
            count[((x >> shift) & 0xFFFF) as usize] += 1;
        }
        let mut pos = 0;
        for c in count.iter_mut() {
            let k = *c;
            *c = pos;
            pos += k;
        }
        for &x in src.iter() {
            let d = ((x >> shift) & 0xFFFF) as usize;
            dst[count[d]] = x;
            count[d] += 1;
        }
        std::mem::swap(&mut src, &mut dst);
    }
}

// counting1024 sorts bare keys < 1024 by rewriting them from their counts.
fn counting1024(xs: &mut [u64]) {
    let mut count = [0usize; 1024];
    for &x in xs.iter() {
        count[x as usize] += 1;
    }
    let mut i = 0;
    for (v, &c) in count.iter().enumerate() {
        xs[i..i + c].fill(v as u64);
        i += c;
    }
}

fn std_sort(xs: &mut [u64]) {
    xs.sort_unstable();
}
fn stable_sort(xs: &mut [u64]) {
    xs.sort();
}
fn copy_only(xs: &mut [u64]) {
    black_box(xs); // the control: same loop, same call, no sort
}

/// ns per call of `copy pristine -> work, then f(work)`. The copy is INSIDE
/// the timed loop on purpose: sorting an already-sorted slice measures
/// nothing. The copy's own cost is measured separately (copy_only) and
/// subtracted by the caller. Best of 3 batches of >= 20 ms each.
fn ns_per_call(work: &mut [u64], pristine: &[u64], f: Sorter) -> f64 {
    let mut batch = |reps: u32| {
        let start = Instant::now();
        for _ in 0..reps {
            work.copy_from_slice(pristine);
            f(black_box(&mut *work));
        }
        start.elapsed()
    };
    let mut reps = 1;
    while batch(reps) < Duration::from_millis(20) {
        reps *= 2;
    }
    (0..3)
        .map(|_| batch(reps).as_nanos() as f64 / f64::from(reps))
        .fold(f64::INFINITY, f64::min)
}

fn label(n: usize) -> String {
    if n >= 1 << 20 {
        format!("{}M", n >> 20)
    } else if n >= 1 << 10 {
        format!("{}K", n >> 10)
    } else {
        n.to_string()
    }
}

/// The first n where `fast` beats `base`, and the smallest n from which it
/// wins at EVERY larger n in the table. `None` = never.
fn crossover(ns: &[usize], fast: &[f64], base: &[f64]) -> (Option<usize>, Option<usize>) {
    let wins = |i: usize| fast[i] < base[i];
    let first = (0..ns.len()).find(|&i| wins(i)).map(|i| ns[i]);
    let tail = (0..ns.len()).rev().take_while(|&i| wins(i)).count();
    let from = (tail > 0).then(|| ns[ns.len() - tail]);
    (first, from)
}

fn verdict(name: &str, opp: &str, ns: &[usize], fast: &[f64], base: &[f64]) {
    let last = ns.len() - 1;
    let big = label(ns[last]);
    match crossover(ns, fast, base) {
        (None, _) => println!(
            "  {name:<12} never beats {opp} in this range (at n={big}: {:.2}x its time)",
            fast[last] / base[last]
        ),
        (Some(first), None) => println!(
            "  {name:<12} first beats {opp} at n={}, but loses again at n={big}",
            label(first)
        ),
        (Some(first), Some(from)) if first == from => println!(
            "  {name:<12} beats {opp} from n={} on (at n={big}: {:.1}x faster)",
            label(from),
            base[last] / fast[last]
        ),
        (Some(first), Some(from)) => println!(
            "  {name:<12} first beats {opp} at n={}, and at every n from {} on (at n={big}: {:.1}x faster)",
            label(first),
            label(from),
            base[last] / fast[last]
        ),
    }
}

fn main() {
    let ns = [
        16,
        64,
        256,
        1 << 10,
        4 << 10,
        16 << 10,
        64 << 10,
        256 << 10,
        1 << 20,
        4 << 20,
    ];
    let max_n = ns[ns.len() - 1];

    // Same seed, same generator, same draws as the Go program.
    // (No GC here: each LSD buffer is freed when the call returns.)
    let mut rng = Pcg::new(1, 2);
    let wide: Vec<u64> = (0..max_n).map(|_| rng.next_u64()).collect();
    let small: Vec<u64> = (0..max_n).map(|_| rng.next_u64() % 1024).collect();
    let mut work = vec![0u64; max_n];
    let mut reference = vec![0u64; max_n];

    // 1. Correctness before any clock is started.
    let cases: [(&[u64], Sorter); 4] = [
        (&wide, lsd8),
        (&wide, lsd16),
        (&small, lsd8),
        (&small, counting1024),
    ];
    let mut ok = true;
    for &n in &ns {
        for &(input, f) in &cases {
            let (r, w) = (&mut reference[..n], &mut work[..n]);
            r.copy_from_slice(&input[..n]);
            r.sort_unstable();
            w.copy_from_slice(&input[..n]);
            f(w);
            ok &= r == w;
        }
    }
    println!(
        "1. lsd8, lsd16 (random keys) and lsd8, counting (keys < 1024) equal sort_unstable at all {} sizes: {ok}\n",
        ns.len()
    );

    // 2. The clock. ns per ELEMENT, copy cost subtracted.
    println!("2. ns/element (copy-only control subtracted; best of 3)");
    println!(
        "{:>6} {:<32} | {:<23} |",
        "", "random u64 keys", "keys < 1024"
    );
    println!(
        "{:>6} {:>13} {:>8} {:>9} | {:>13} {:>9} | {:>5}",
        "n", "sort_unstable", "lsd8", "lsd16", "sort_unstable", "counting", "copy"
    );
    let (mut std_w, mut l8, mut l16, mut std_s, mut cnt) = (vec![], vec![], vec![], vec![], vec![]);
    for &n in &ns {
        let w = &mut work[..n];
        let ctrl = ns_per_call(w, &wide[..n], copy_only);
        let mut per = |input: &[u64], f: Sorter| (ns_per_call(w, &input[..n], f) - ctrl) / n as f64;
        std_w.push(per(&wide, std_sort));
        l8.push(per(&wide, lsd8));
        l16.push(per(&wide, lsd16));
        std_s.push(per(&small, std_sort));
        cnt.push(per(&small, counting1024));
        let i = std_w.len() - 1;
        println!(
            "{:>6} {:>13.2} {:>8.2} {:>9.2} | {:>13.2} {:>9.2} | {:>5.2}",
            label(n),
            std_w[i],
            l8[i],
            l16[i],
            std_s[i],
            cnt[i],
            ctrl / n as f64
        );
    }

    // 3. The verdict, computed from the table above.
    println!("\n3. Crossover (from this run's numbers)");
    verdict("lsd8", "sort_unstable", &ns, &l8, &std_w);
    verdict("lsd16", "sort_unstable", &ns, &l16, &std_w);
    verdict("counting", "sort_unstable", &ns, &cnt, &std_s);
    println!(
        "  fixed cost per call: lsd8 clears {} counters, lsd16 {}, counting {}",
        8 * 256,
        4 * (1 << 16),
        1024
    );

    // 4. Allocations per call (deterministic: counted by the global allocator).
    println!("\n4. Heap allocations per call (counting #[global_allocator]): allocs / bytes");
    println!("{:<24} {:>14} {:>16}", "", "n=16", "n=1M");
    let rows: [(&str, Sorter); 5] = [
        ("sort_unstable", std_sort),
        ("sort (stable)", stable_sort),
        ("lsd8", lsd8),
        ("lsd16", lsd16),
        ("counting (keys < 1024)", counting1024),
    ];
    for (name, f) in rows {
        let mut row = format!("{name:<24}");
        for (n, width) in [(16, 14), (1 << 20, 16)] {
            let input = if name.starts_with("counting") {
                &small
            } else {
                &wide
            };
            let w = &mut work[..n];
            w.copy_from_slice(&input[..n]);
            let (a, b) = allocs_of(|| f(w));
            row += &format!(" {:>width$}", format!("{a} / {b}"));
        }
        println!("{row}");
    }
}
```

**Rust sample output:**

```
1. lsd8, lsd16 (random keys) and lsd8, counting (keys < 1024) equal sort_unstable at all 10 sizes: true

2. ns/element (copy-only control subtracted; best of 3)
       random u64 keys                  | keys < 1024             |
     n sort_unstable     lsd8     lsd16 | sort_unstable  counting |  copy
    16          1.74    54.58   5555.63 |          1.78     40.46 |  0.13
    64          2.91    17.09   1436.80 |          2.65      9.67 |  0.09
   256          3.74     9.10    364.05 |          3.55      2.92 |  0.07
    1K          4.41     5.95    104.98 |          4.40      1.54 |  0.08
    4K          5.17     5.34     28.87 |          5.06      0.82 |  0.08
   16K          6.03     6.32     12.57 |          5.58      0.51 |  0.11
   64K          7.10     5.95      8.94 |          4.87      0.37 |  0.11
  256K          8.79     6.15      8.35 |          4.73      0.34 |  0.11
    1M         10.19     7.04      8.45 |          4.62      0.40 |  0.15
    4M         11.04     7.91      8.56 |          4.75      0.37 |  0.19

3. Crossover (from this run's numbers)
  lsd8         beats sort_unstable from n=64K on (at n=4M: 1.4x faster)
  lsd16        beats sort_unstable from n=256K on (at n=4M: 1.3x faster)
  counting     beats sort_unstable from n=256 on (at n=4M: 12.9x faster)
  fixed cost per call: lsd8 clears 2048 counters, lsd16 262144, counting 1024

4. Heap allocations per call (counting #[global_allocator]): allocs / bytes
                                   n=16             n=1M
sort_unstable                     0 / 0            0 / 0
sort (stable)                     0 / 0      1 / 8000000
lsd8                            1 / 128      1 / 8388608
lsd16                           1 / 128      1 / 8388608
counting (keys < 1024)            0 / 0            0 / 0
```

> **Go vs Rust:** Same keys (the PCG port, seed 1, 2) and the same LSD and counting code, and those run at similar speed in both languages (`lsd8` about 7–9 ns/element at 1M–4M in each); what moves the crossover is the opponent. `sort_unstable` (ipnsort) ran about 5–6x faster than Go's `slices.Sort` on random keys (about 11 vs about 63 ns/element at 4M; why is not measured here), so across 6 runs Rust's `lsd8` took over only at 16K or 64K instead of Go's 1K, and won by only 1.2–1.5x at 4M (one noisy run: a tie) instead of about 8x; counting sort still wins from n = 256 in both, by about 12x at 4M instead of about 50x.
> Allocations differ too: Rust's `lsd16` makes 1 where Go's makes 5, because Rust keeps a 512 KiB local array on the stack while Go's compiler moves anything above `MaxStackVarSize` = 128 KiB to the heap (read in `cmd/compile/internal/ir/cfg.go`, not run), and Rust's stable `sort` (driftsort) allocates 1,000,000 elements at n = 1M where Go's `slices.SortStableFunc` allocates nothing.

**Complexity:** LSD-8 is Θ(8·(n + 256)) on 64-bit keys, LSD-16 Θ(4·(n + 65,536)), counting sort Θ(n + 1024), against pdqsort's Θ(n log n) · extra space: one n-element buffer per LSD call (1 allocation; Go's `lsd16` makes 5, because its 512 KiB histogram is too big for the stack) vs 0 for `slices.Sort` and `slices.SortStableFunc` · measured over 5 runs here: `lsd8` wins from n = 1K on, `lsd16` from 16K, counting from 256; at n = 4M `lsd8` is about 7–9x and counting about 40–50x faster, while at n = 16 `slices.Sort` is more than 20x faster than `lsd8`.

---

## 13. Real keys: timestamps, IDs and the passes you can skip

`🔴 hard` · *Applied / pass skipping*

Real keys are not uniform 64-bit noise. A day of Unix-nanosecond timestamps shares its top two bytes;
a million database IDs fit in 21 bits, so five of their eight bytes are zero. An LSD pass over a byte
that is the same in every key reads and rewrites the whole array and changes nothing. The fix is
cheap: read the input **once** and build all eight digit histograms together (counts do not depend
on order, so they stay valid for every pass), then skip each pass whose histogram has a single
non-empty bucket. Time becomes Θ((1 + p)·n + 8·256) for the p passes actually needed, and the
program counts the reads and writes that it saves before it times them.

**Steps:**

1. Build three datasets of 2²⁰ keys from a fixed seed: one day of timestamps, shuffled IDs with
   about 10% gaps, and uniform `uint64`; print how many distinct values each byte takes.
2. Sort each with textbook `lsd8` and with `lsdSkip`, which counts every scan it makes, and check
   both against `slices.Sort`.
3. Compare the counted reads and writes (exact) with the clock (a sample), including the uniform
   keys, where there is nothing to skip.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"time"
)

// Real keys are not uniform 64-bit noise. Timestamps from one day share
// their top bytes; database IDs below 2^21 have five zero bytes. An LSD pass
// over a digit that is the SAME for every key is a full read + write of the
// array that changes nothing. The trick to skip it:
//
//  1. read the input ONCE and build all eight digit histograms at the same
//     time. Counts do not depend on the order of the keys, so histograms
//     taken from the input stay valid for every later pass;
//  2. a digit whose histogram has a single non-empty bucket (count == n)
//     cannot reorder anything: skip that pass.

// stats are counted per scan of the array (+n per loop over n keys), so the
// counting costs nothing measurable and is identical in every run.
type stats struct {
	reads, writes, passes, skipped int
}

// lsd8 is the textbook version: 8 passes, each one reads the array twice
// (count, then scatter) and writes it once.
func lsd8(xs []uint64) (st stats) {
	n := len(xs)
	buf := make([]uint64, n)
	src, dst := xs, buf
	for shift := uint(0); shift < 64; shift += 8 {
		var count [256]int
		for _, x := range src {
			count[byte(x>>shift)]++
		}
		pos := 0
		for d, c := range count {
			count[d] = pos
			pos += c
		}
		for _, x := range src {
			d := byte(x >> shift)
			dst[count[d]] = x
			count[d]++
		}
		st.reads += 2 * n
		st.writes += n
		st.passes++
		src, dst = dst, src
	}
	return st
}

// lsdSkip builds all eight histograms in one read, then runs only the passes
// whose digit actually varies.
func lsdSkip(xs []uint64) (st stats) {
	n := len(xs)
	if n < 2 {
		return st
	}
	var hist [8][256]int // 16 KiB, on the stack
	for _, x := range xs {
		for d := range 8 {
			// byte(...) is 0..255, so the compiler drops the bounds check
			// on hist[d][...]
			hist[d][byte(x>>(8*d))]++
		}
	}
	st.reads += n
	buf := make([]uint64, n)
	src, dst := xs, buf
	for d := range 8 {
		shift := uint(8 * d)
		count := &hist[d]
		if count[byte(xs[0]>>shift)] == n { // one bucket holds every key
			st.skipped++
			continue
		}
		pos := 0
		for b, c := range count {
			count[b] = pos
			pos += c
		}
		for _, x := range src {
			b := byte(x >> shift)
			dst[count[b]] = x
			count[b]++
		}
		st.reads += n
		st.writes += n
		st.passes++
		src, dst = dst, src
	}
	if st.passes%2 == 1 { // an odd number of passes ends in buf
		copy(xs, src)
		st.writes += n
	}
	return st
}

func stdSort(xs []uint64) stats { slices.Sort(xs); return stats{} }
func copyOnly([]uint64) stats   { return stats{} }

// nsPerCall: copy a pristine input into work INSIDE the timed loop (sorting
// sorted data measures nothing), best of 3 batches of >= 20 ms. The caller
// subtracts the copy-only control.
func nsPerCall(work, pristine []uint64, f func([]uint64) stats) float64 {
	batch := func(reps int) time.Duration {
		start := time.Now()
		for range reps {
			copy(work, pristine)
			f(work)
		}
		return time.Since(start)
	}
	reps := 1
	for batch(reps) < 20*time.Millisecond {
		reps *= 2
	}
	best := math.Inf(1)
	for range 3 {
		best = min(best, float64(batch(reps).Nanoseconds())/float64(reps))
	}
	return best
}

func main() {
	const n = 1 << 20
	rng := rand.New(rand.NewPCG(1, 2))

	// (a) Unix-nanosecond timestamps, all within 2026-11-09 UTC.
	base := uint64(time.Date(2026, 11, 9, 0, 0, 0, 0, time.UTC).UnixNano())
	stamps := make([]uint64, n)
	for i := range stamps {
		stamps[i] = base + rng.Uint64()%(86_400*1_000_000_000)
	}
	// (b) IDs: 1, 2, 3, ... with about 10% of them deleted, then shuffled.
	ids := make([]uint64, 0, n)
	for id := uint64(1); len(ids) < n; id++ {
		if rng.Uint64()%10 != 0 {
			ids = append(ids, id)
		}
	}
	for i := n - 1; i > 0; i-- { // Fisher-Yates
		j := rng.Uint64() % uint64(i+1)
		ids[i], ids[j] = ids[j], ids[i]
	}
	// (c) uniform 64-bit keys.
	wide := make([]uint64, n)
	for i := range wide {
		wide[i] = rng.Uint64()
	}

	sets := []struct {
		name string
		keys []uint64
	}{{"timestamps", stamps}, {"IDs", ids}, {"uniform", wide}}

	// 1. What the keys look like: distinct values of each byte.
	fmt.Printf("n = %d keys per dataset\n\n", n)
	fmt.Println("1. Non-empty buckets per byte (byte 7 = most significant)")
	fmt.Printf("%-11s %16s %16s   %s\n", "dataset", "min", "max", "b7  b6  b5  b4  b3  b2  b1  b0")
	for _, s := range sets {
		var seen [8][256]bool
		for _, x := range s.keys {
			for d := range 8 {
				seen[d][byte(x>>(8*d))] = true
			}
		}
		row := fmt.Sprintf("%-11s %016x %016x  ", s.name, slices.Min(s.keys), slices.Max(s.keys))
		for d := 7; d >= 0; d-- {
			k := 0
			for _, b := range seen[d] {
				if b {
					k++
				}
			}
			row += fmt.Sprintf("%4d", k)
		}
		fmt.Println(row)
	}

	// 2. Counted work, and correctness, before any clock.
	fmt.Println("\n2. Counted work (deterministic)")
	fmt.Printf("%-11s %7s %8s | %12s %12s | %13s %13s\n", "dataset", "passes", "skipped",
		"naive reads", "skip reads", "naive writes", "skip writes")
	work := make([]uint64, n)
	ref := make([]uint64, n)
	ok := true
	for _, s := range sets {
		copy(ref, s.keys)
		slices.Sort(ref)
		copy(work, s.keys)
		a := lsd8(work)
		ok = ok && slices.Equal(work, ref)
		copy(work, s.keys)
		b := lsdSkip(work)
		ok = ok && slices.Equal(work, ref)
		fmt.Printf("%-11s %7d %8d | %12d %12d | %13d %13d\n", s.name, b.passes, b.skipped,
			a.reads, b.reads, a.writes, b.writes)
		fmt.Printf("%-11s %7s %8s | %11.1fn %11.1fn | %12.1fn %12.1fn\n", "", "", "",
			float64(a.reads)/n, float64(b.reads)/n, float64(a.writes)/n, float64(b.writes)/n)
	}
	fmt.Printf("lsd8 and lsdSkip equal slices.Sort on all three datasets: %v\n", ok)

	// 3. What the skipped passes are worth on the clock.
	fmt.Println("\n3. ns/element (copy-only control subtracted; best of 3)")
	fmt.Printf("%-11s %8s %8s %12s   %s\n", "dataset", "lsd8", "lsdSkip", "slices.Sort", "lsd8 / lsdSkip")
	for _, s := range sets {
		ctrl := nsPerCall(work, s.keys, copyOnly)
		per := func(f func([]uint64) stats) float64 { return (nsPerCall(work, s.keys, f) - ctrl) / n }
		a, b, c := per(lsd8), per(lsdSkip), per(stdSort)
		fmt.Printf("%-11s %8.2f %8.2f %12.2f   %.2fx\n", s.name, a, b, c, a/b)
	}
}
```

**Sample output:**

```
n = 1048576 keys per dataset

1. Non-empty buckets per byte (byte 7 = most significant)
dataset                  min              max   b7  b6  b5  b4  b3  b2  b1  b0
timestamps  18e637630a7e23fc 18e685f794b06bb6     1   1  79 256 256 256 256 256
IDs         0000000000000001 000000000011c7a3     1   1   1   1   1  18 256 256
uniform     00000d3a5ba51b7e ffffff6040014b64   256 256 256 256 256 256 256 256

2. Counted work (deterministic)
dataset      passes  skipped |  naive reads   skip reads |  naive writes   skip writes
timestamps        6        2 |     16777216      7340032 |       8388608       6291456
                             |        16.0n         7.0n |          8.0n          6.0n
IDs               3        5 |     16777216      4194304 |       8388608       4194304
                             |        16.0n         4.0n |          8.0n          4.0n
uniform           8        0 |     16777216      9437184 |       8388608       8388608
                             |        16.0n         9.0n |          8.0n          8.0n
lsd8 and lsdSkip equal slices.Sort on all three datasets: true

3. ns/element (copy-only control subtracted; best of 3)
dataset         lsd8  lsdSkip  slices.Sort   lsd8 / lsdSkip
timestamps      9.55     6.14        57.46   1.56x
IDs            12.63     5.01        56.68   2.52x
uniform         6.88     7.12        57.83   0.97x
```

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// Real keys are not uniform 64-bit noise. Timestamps from one day share
// their top bytes; database IDs below 2^21 have five zero bytes. An LSD pass
// over a digit that is the SAME for every key is a full read + write of the
// array that changes nothing. The trick to skip it:
//
//  1. read the input ONCE and build all eight digit histograms at the same
//     time. Counts do not depend on the order of the keys, so histograms
//     taken from the input stay valid for every later pass;
//  2. a digit whose histogram has a single non-empty bucket (count == n)
//     cannot reorder anything: skip that pass.

// ---- Go's math/rand/v2 PCG (PCG-DXSM), so the keys are Go's keys ----
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
}

/// Counted per scan of the array (+n per loop over n keys), so counting costs
/// nothing measurable and is identical in every run.
#[derive(Default, Clone, Copy)]
struct Stats {
    reads: usize,
    writes: usize,
    passes: usize,
    skipped: usize,
}

type Sorter = fn(&mut [u64]) -> Stats;

/// The textbook version: 8 passes, each reads the array twice (count, then
/// scatter) and writes it once.
fn lsd8(xs: &mut [u64]) -> Stats {
    let n = xs.len();
    let mut st = Stats::default();
    let mut buf = vec![0u64; n];
    let (mut src, mut dst): (&mut [u64], &mut [u64]) = (xs, &mut buf);
    for shift in (0..64).step_by(8) {
        let mut count = [0usize; 256];
        for &x in src.iter() {
            count[((x >> shift) & 0xFF) as usize] += 1;
        }
        let mut pos = 0;
        for c in count.iter_mut() {
            let k = *c;
            *c = pos;
            pos += k;
        }
        for &x in src.iter() {
            let b = ((x >> shift) & 0xFF) as usize;
            dst[count[b]] = x;
            count[b] += 1;
        }
        st.reads += 2 * n;
        st.writes += n;
        st.passes += 1;
        std::mem::swap(&mut src, &mut dst);
    }
    st
}

/// All eight histograms in one read, then only the passes whose digit varies.
fn lsd_skip(xs: &mut [u64]) -> Stats {
    let n = xs.len();
    let mut st = Stats::default();
    if n < 2 {
        return st;
    }
    // 8 x 256 counters = 16 KiB, a plain array on the stack. `to_le_bytes`
    // splits a key into its 8 digits; each digit is a u8, so indexing a
    // 256-entry array with it can never be out of bounds and the compiler
    // emits no bounds check for it.
    let mut hist = [[0usize; 256]; 8];
    for &x in xs.iter() {
        for (h, b) in hist.iter_mut().zip(x.to_le_bytes()) {
            h[usize::from(b)] += 1;
        }
    }
    st.reads += n;
    let first = xs[0].to_le_bytes();
    let mut buf = vec![0u64; n];
    let (mut src, mut dst): (&mut [u64], &mut [u64]) = (&mut *xs, &mut buf);
    for (pass, count) in hist.iter_mut().enumerate() {
        if count[usize::from(first[pass])] == n {
            // one bucket holds every key
            st.skipped += 1;
            continue;
        }
        let mut pos = 0;
        for c in count.iter_mut() {
            let k = *c;
            *c = pos;
            pos += k;
        }
        // A shift, not `x.to_le_bytes()[pass]`: with the byte array indexed by
        // a run-time `pass`, this loop measured about 2x slower on uniform
        // keys while this example was written (why was not investigated).
        let shift = 8 * pass;
        for &x in src.iter() {
            let b = ((x >> shift) & 0xFF) as usize;
            dst[count[b]] = x;
            count[b] += 1;
        }
        st.reads += n;
        st.writes += n;
        st.passes += 1;
        std::mem::swap(&mut src, &mut dst);
    }
    // src/dst are no longer used, so their borrows of xs and buf have ended.
    if st.passes % 2 == 1 {
        // an odd number of passes ends in buf
        xs.copy_from_slice(&buf);
        st.writes += n;
    }
    st
}

fn std_sort(xs: &mut [u64]) -> Stats {
    xs.sort_unstable();
    Stats::default()
}
fn copy_only(xs: &mut [u64]) -> Stats {
    black_box(xs);
    Stats::default()
}

/// Copy a pristine input into work INSIDE the timed loop (sorting sorted data
/// measures nothing), best of 3 batches of >= 20 ms. The caller subtracts the
/// copy-only control.
fn ns_per_call(work: &mut [u64], pristine: &[u64], f: Sorter) -> f64 {
    let mut batch = |reps: u32| {
        let start = Instant::now();
        for _ in 0..reps {
            work.copy_from_slice(pristine);
            black_box(f(black_box(&mut *work)));
        }
        start.elapsed()
    };
    let mut reps = 1;
    while batch(reps) < Duration::from_millis(20) {
        reps *= 2;
    }
    (0..3)
        .map(|_| batch(reps).as_nanos() as f64 / f64::from(reps))
        .fold(f64::INFINITY, f64::min)
}

fn main() {
    const N: usize = 1 << 20;
    let mut rng = Pcg::new(1, 2);

    // (a) Unix-nanosecond timestamps, all within 2026-11-09 UTC. Std has no
    // calendar, so the base is a constant: Go's
    // time.Date(2026, 11, 9, 0, 0, 0, 0, time.UTC).UnixNano().
    const BASE: u64 = 1_794_182_400_000_000_000;
    let stamps: Vec<u64> = (0..N)
        .map(|_| BASE + rng.next_u64() % (86_400 * 1_000_000_000))
        .collect();
    // (b) IDs: 1, 2, 3, ... with about 10% of them deleted, then shuffled.
    let mut ids = Vec::with_capacity(N);
    let mut id = 1u64;
    while ids.len() < N {
        if !rng.next_u64().is_multiple_of(10) {
            ids.push(id);
        }
        id += 1;
    }
    for i in (1..N).rev() {
        // Fisher-Yates, drawing exactly what the Go program draws
        let j = (rng.next_u64() % (i as u64 + 1)) as usize;
        ids.swap(i, j);
    }
    // (c) uniform 64-bit keys.
    let wide: Vec<u64> = (0..N).map(|_| rng.next_u64()).collect();

    let sets: [(&str, &[u64]); 3] = [("timestamps", &stamps), ("IDs", &ids), ("uniform", &wide)];

    // 1. What the keys look like: distinct values of each byte.
    println!("n = {N} keys per dataset\n");
    println!("1. Non-empty buckets per byte (byte 7 = most significant)");
    println!(
        "{:<11} {:>16} {:>16}   b7  b6  b5  b4  b3  b2  b1  b0",
        "dataset", "min", "max"
    );
    for &(name, keys) in &sets {
        let mut seen = [[false; 256]; 8];
        for &x in keys {
            for (s, b) in seen.iter_mut().zip(x.to_le_bytes()) {
                s[usize::from(b)] = true;
            }
        }
        let (lo, hi) = (keys.iter().min().unwrap(), keys.iter().max().unwrap());
        let mut row = format!("{name:<11} {lo:016x} {hi:016x}  ");
        for s in seen.iter().rev() {
            row += &format!("{:>4}", s.iter().filter(|&&b| b).count());
        }
        println!("{row}");
    }

    // 2. Counted work, and correctness, before any clock.
    println!("\n2. Counted work (deterministic)");
    println!(
        "{:<11} {:>7} {:>8} | {:>12} {:>12} | {:>13} {:>13}",
        "dataset", "passes", "skipped", "naive reads", "skip reads", "naive writes", "skip writes"
    );
    let mut work = vec![0u64; N];
    let mut ok = true;
    for &(name, keys) in &sets {
        let mut reference = keys.to_vec();
        reference.sort_unstable();
        work.copy_from_slice(keys);
        let a = lsd8(&mut work);
        ok &= work == reference;
        work.copy_from_slice(keys);
        let b = lsd_skip(&mut work);
        ok &= work == reference;
        println!(
            "{name:<11} {:>7} {:>8} | {:>12} {:>12} | {:>13} {:>13}",
            b.passes, b.skipped, a.reads, b.reads, a.writes, b.writes
        );
        let per_n = |v: usize| format!("{:.1}n", v as f64 / N as f64);
        println!(
            "{:<11} {:>7} {:>8} | {:>12} {:>12} | {:>13} {:>13}",
            "",
            "",
            "",
            per_n(a.reads),
            per_n(b.reads),
            per_n(a.writes),
            per_n(b.writes)
        );
    }
    println!("lsd8 and lsd_skip equal sort_unstable on all three datasets: {ok}");

    // 3. What the skipped passes are worth on the clock.
    println!("\n3. ns/element (copy-only control subtracted; best of 3)");
    println!(
        "{:<11} {:>8} {:>8} {:>14}   lsd8 / lsd_skip",
        "dataset", "lsd8", "lsd_skip", "sort_unstable"
    );
    for &(name, keys) in &sets {
        let ctrl = ns_per_call(&mut work, keys, copy_only);
        let mut per = |f: Sorter| (ns_per_call(&mut work, keys, f) - ctrl) / N as f64;
        let (a, b, c) = (per(lsd8), per(lsd_skip), per(std_sort));
        println!("{name:<11} {a:>8.2} {b:>8.2} {c:>14.2}   {:.2}x", a / b);
    }
}
```

**Rust sample output:**

```
n = 1048576 keys per dataset

1. Non-empty buckets per byte (byte 7 = most significant)
dataset                  min              max   b7  b6  b5  b4  b3  b2  b1  b0
timestamps  18e637630a7e23fc 18e685f794b06bb6     1   1  79 256 256 256 256 256
IDs         0000000000000001 000000000011c7a3     1   1   1   1   1  18 256 256
uniform     00000d3a5ba51b7e ffffff6040014b64   256 256 256 256 256 256 256 256

2. Counted work (deterministic)
dataset      passes  skipped |  naive reads   skip reads |  naive writes   skip writes
timestamps        6        2 |     16777216      7340032 |       8388608       6291456
                             |        16.0n         7.0n |          8.0n          6.0n
IDs               3        5 |     16777216      4194304 |       8388608       4194304
                             |        16.0n         4.0n |          8.0n          4.0n
uniform           8        0 |     16777216      9437184 |       8388608       8388608
                             |        16.0n         9.0n |          8.0n          8.0n
lsd8 and lsd_skip equal sort_unstable on all three datasets: true

3. ns/element (copy-only control subtracted; best of 3)
dataset         lsd8 lsd_skip  sort_unstable   lsd8 / lsd_skip
timestamps     11.22     4.75           9.83   2.36x
IDs            16.96     3.55           9.75   4.77x
uniform         7.06     6.03           9.88   1.17x
```

> **Go vs Rust:** Sections 1–2 are identical to Go's (same PCG keys, same counted reads and writes; only the stdlib label differs). The clock tells a different story because the opponent is different: `sort_unstable` ran about 10 ns/element on all three datasets against about 57 for Go's `slices.Sort`, so Rust's textbook `lsd8` (7 on uniform keys, but 10–17 on timestamps and IDs across 4 runs) ties or loses against the stdlib on the skewed data, and only `lsd_skip` (about 3.5–6.6) beats it on all three; skipping also gained about 1.15x on uniform keys here, where Go's gained nothing.
> The eight histograms are a `[[usize; 256]; 8]` filled by zipping each key's `to_le_bytes()` with them; the `u8` digit index leaves no bounds check (checked in the assembly of a stand-alone copy of that loop). The scatter loop uses a shift instead, because `to_le_bytes()[pass]` with a run-time `pass` measured about 2x slower there.

**Complexity:** textbook LSD-8 is Θ(8·(n + 256)) whatever the keys: 16n reads and 8n writes · with one-pass histograms and skipping, Θ((1 + p)·n + 8·256) for the p passes needed: timestamps need 6 of 8 (7n reads, 6n writes), IDs 3 (4n reads, 4n writes, including the copy back after an odd number of passes), uniform keys all 8 (9n reads, no write saved) · space Θ(n) buffer + 16 KiB of histograms · on the clock over 5 runs, skipping was 1.3–2.6x faster than `lsd8` on timestamps and about 2.5–3.9x on IDs, and gained nothing on uniform keys (0.66–1.0x, sometimes slower); both LSD versions beat `slices.Sort` on all three datasets.

---

## 14. Shared prefixes: MSD radix vs comparison sort

`🔴 hard` · *Strings measured*

A comparison sort sees a string only through `a < b?`, and every answer starts again at byte 0: two
URLs sharing a 33-byte prefix cost at least 34 byte compares each time they meet, and they meet
O(log n) times. MSD radix sort reads byte d of a string only while it still shares a bucket with
another string, so without a cutoff it reads exactly the **distinguishing prefix** D, plus one
end-of-string digit for strings that are a prefix of another. That is Θ(D + R·calls) byte work against
Θ(n log n) comparisons that each cost up to the shared-prefix length. This program counts both on
random short strings and on URLs, and then times them, because bytes read is not the only cost: every
MSD level also moves the strings to a buffer and back.

**Steps:**

1. Build 200,000 random 8–16-byte lowercase strings and 200,000 URLs with a 33-byte common prefix;
   compute D directly from the sorted order (the longest common prefix with each neighbour).
2. Count the bytes read by MSD (no cutoff, cutoff 16, cutoff 16 with single-bucket levels skipped)
   and by `slices.SortFunc` with a comparator that counts every byte up to the first difference.
3. Check every result against `slices.Sort`, then time `slices.Sort` against both MSD-16 versions on
   both datasets.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"time"
)

// A comparison sort sees a string only through "a < b?", and every answer
// starts at byte 0. Two URLs that share 33 bytes cost at least 34 byte
// compares EVERY time they meet, and they meet O(log n) times. MSD radix
// sort reads byte d of a string only while it still shares a bucket with
// another string -- so it reads each string's distinguishing prefix about
// once. This program counts both, then times them.

// msd is an out-of-place MSD radix sort with an insertion-sort cutoff.
// Bucket 0 is "string ended here"; byte b goes to bucket b+1, so "she" sorts
// before "shells". With skip set, a level where every string has the same
// byte is not distributed at all (the pass skipping of example 13).
type msd struct {
	cutoff      int
	skip        bool
	aux         []string // the n-element buffer
	dig         []int    // each string's digit at the current depth, read once
	reads       int      // bytes read by the radix passes
	insertBytes int      // bytes compared inside insertion sort
	calls       int      // radix calls (each clears 258 counters)
	moves       int      // string headers written (to aux and back, swaps)
}

func (m *msd) sort(a []string, d int) {
	if len(a) <= max(m.cutoff, 1) {
		m.insertion(a, d)
		return
	}
	m.calls++
	var count [258]int
	for i, s := range a {
		c := 0 // end of string
		if d < len(s) {
			c = int(s[d]) + 1
		}
		m.dig[i] = c
		count[c+1]++
		m.reads++
	}
	if c := m.dig[0]; m.skip && count[c+1] == len(a) { // one bucket holds all
		if c > 0 { // (c == 0: all ended here, so all are equal: done)
			m.sort(a, d+1)
		}
		return
	}
	for r := range 257 { // counts -> start positions
		count[r+1] += count[r]
	}
	for i, s := range a { // stable distribution into aux
		c := m.dig[i]
		m.aux[count[c]] = s
		count[c]++
	}
	copy(a, m.aux[:len(a)])
	m.moves += 2 * len(a)
	// Now count[c] is the END of bucket c. Bucket 0 (strings that ended) is
	// already in its final order: they are all equal. Recurse on the rest.
	for c := 1; c < 257; c++ {
		m.sort(a[count[c-1]:count[c]], d+1)
	}
}

// insertion sorts strings that all share their first d bytes, so it compares
// from byte d on and counts every byte it looks at.
func (m *msd) insertion(a []string, d int) {
	for i := 1; i < len(a); i++ {
		for j := i; j > 0 && m.less(a[j], a[j-1], d); j-- {
			a[j], a[j-1] = a[j-1], a[j]
			m.moves += 2
		}
	}
}

func (m *msd) less(s, t string, d int) bool {
	for ; d < len(s) && d < len(t); d++ {
		m.insertBytes++
		if s[d] != t[d] {
			return s[d] < t[d]
		}
	}
	return len(s) < len(t)
}

func msdSort(a []string, cutoff int, skip bool) *msd {
	m := &msd{cutoff: cutoff, skip: skip, aux: make([]string, len(a)), dig: make([]int, len(a))}
	m.sort(a, 0)
	return m
}

// cmpCounting is a byte-wise three-way compare that counts what it reads:
// every byte up to and including the first difference.
func cmpCounting(bytes, calls *int) func(a, b string) int {
	return func(a, b string) int {
		*calls++
		n := min(len(a), len(b))
		for i := range n {
			*bytes++
			if a[i] != b[i] {
				return int(a[i]) - int(b[i])
			}
		}
		return len(a) - len(b)
	}
}

// distinguishing returns D, the total distinguishing prefix: for each string,
// one byte more than its longest common prefix with any other string (capped
// at its length), and E, how many strings are a prefix of another string or
// equal to one (MSD reads one extra "end of string" digit for those).
func distinguishing(sorted []string) (D, E int) {
	for i, s := range sorted {
		l := 0
		if i > 0 {
			l = lcp(sorted[i-1], s)
		}
		if i+1 < len(sorted) {
			l = max(l, lcp(s, sorted[i+1]))
		}
		if l >= len(s) {
			D += len(s)
			E++
		} else {
			D += l + 1
		}
	}
	return D, E
}

// nsPerCall: copy the pristine order INSIDE the timed loop (a []string copy
// moves 16-byte headers, not the bytes), best of 3 batches of >= 20 ms.
func nsPerCall(work, pristine []string, f func([]string)) float64 {
	batch := func(reps int) time.Duration {
		start := time.Now()
		for range reps {
			copy(work, pristine)
			f(work)
		}
		return time.Since(start)
	}
	reps := 1
	for batch(reps) < 20*time.Millisecond {
		reps *= 2
	}
	best := math.Inf(1)
	for range 3 {
		best = min(best, float64(batch(reps).Nanoseconds())/float64(reps))
	}
	return best
}

func main() {
	const n = 200_000
	const cutoff = 16
	rng := rand.New(rand.NewPCG(1, 2))

	// (a) random lowercase strings, 8..16 bytes.
	random := make([]string, n)
	for i := range random {
		b := make([]byte, 8+rng.Uint64()%9)
		for j := range b {
			b[j] = 'a' + byte(rng.Uint64()%26)
		}
		random[i] = string(b)
	}
	// (b) URLs sharing a 33-byte prefix.
	urls := make([]string, n)
	for i := range urls {
		urls[i] = fmt.Sprintf("https://example.com/api/v1/users/%06d/orders/%d",
			rng.Uint64()%1_000_000, rng.Uint64()%100_000_000)
	}

	sets := []struct {
		name string
		keys []string
	}{{"(a) random 8-16 lowercase", random}, {"(b) URLs", urls}}

	work := make([]string, n)
	allOK := true
	fmt.Printf("n = %d strings per dataset, MSD cutoff = %d\n", n, cutoff)
	for _, s := range sets {
		ref := slices.Clone(s.keys)
		slices.Sort(ref)
		total := 0
		for _, k := range s.keys {
			total += len(k)
		}
		D, E := distinguishing(ref)

		copy(work, s.keys)
		m0 := msdSort(work, 0, false)
		allOK = allOK && slices.Equal(work, ref)
		copy(work, s.keys)
		mc := msdSort(work, cutoff, false)
		allOK = allOK && slices.Equal(work, ref)
		copy(work, s.keys)
		ms := msdSort(work, cutoff, true)
		allOK = allOK && slices.Equal(work, ref)
		var cb, cc int
		copy(work, s.keys)
		slices.SortFunc(work, cmpCounting(&cb, &cc))
		allOK = allOK && slices.Equal(work, ref)

		fmt.Printf("\n%s\n", s.name)
		fmt.Printf("  avg length %.1f bytes; common prefix of all keys: %d bytes\n",
			float64(total)/n, lcp(ref[0], ref[n-1]))
		fmt.Printf("  distinguishing prefix D = %d bytes (%.2f per string)\n", D, float64(D)/n)
		fmt.Printf("  %-16s %11s %8s %10s\n", "", "bytes read", "calls", "moves")
		row := func(name string, m *msd, note string) {
			fmt.Printf("  %-16s %11d %8d %10d   %s\n", name, m.reads+m.insertBytes, m.calls, m.moves, note)
		}
		row("MSD, no cutoff", m0, fmt.Sprintf("= D + E (E = %d): %v", E, m0.reads == D+E))
		row("MSD-16", mc, fmt.Sprintf("radix %d + insertion %d", mc.reads, mc.insertBytes))
		row("MSD-16 + skip", ms, "same bytes, levels with one bucket not moved")
		fmt.Printf("  %-16s %11d %8s %10s   %d comparisons x %.1f bytes\n", "slices.SortFunc", cb, "", "", cc, float64(cb)/float64(cc))
		fmt.Printf("  bytes: slices.SortFunc / MSD-16 = %.1fx;  moves: MSD-16 / MSD-16 + skip = %.1fx\n",
			float64(cb)/float64(mc.reads+mc.insertBytes), float64(mc.moves)/float64(ms.moves))
	}
	fmt.Printf("\nall sorts equal slices.Sort on both datasets: %v\n", allOK)

	fmt.Println("\nns/element (copy-only control subtracted; best of 3)")
	fmt.Printf("%-27s %12s %8s %14s\n", "dataset", "slices.Sort", "MSD-16", "MSD-16 + skip")
	for _, s := range sets {
		ctrl := nsPerCall(work, s.keys, func([]string) {})
		std := (nsPerCall(work, s.keys, func(a []string) { slices.Sort(a) }) - ctrl) / n
		rad := (nsPerCall(work, s.keys, func(a []string) { msdSort(a, cutoff, false) }) - ctrl) / n
		skp := (nsPerCall(work, s.keys, func(a []string) { msdSort(a, cutoff, true) }) - ctrl) / n
		fmt.Printf("%-27s %12.1f %8.1f %14.1f\n", s.name, std, rad, skp)
	}
}

// lcp is the length of the longest common prefix of s and t.
func lcp(s, t string) int {
	i := 0
	for i < len(s) && i < len(t) && s[i] == t[i] {
		i++
	}
	return i
}
```

**Sample output:**

```
n = 200000 strings per dataset, MSD cutoff = 16

(a) random 8-16 lowercase
  avg length 12.0 bytes; common prefix of all keys: 0 bytes
  distinguishing prefix D = 873959 bytes (4.37 per string)
                    bytes read    calls      moves
  MSD, no cutoff        873959    52728    1747918   = D + E (E = 0): true
  MSD-16               1265249     1946    2194252   radix 622704 + insertion 642545
  MSD-16 + skip        1265249     1946    2194252   same bytes, levels with one bucket not moved
  slices.SortFunc      8952726                       3758353 comparisons x 2.4 bytes
  bytes: slices.SortFunc / MSD-16 = 7.1x;  moves: MSD-16 / MSD-16 + skip = 1.0x

(b) URLs
  avg length 54.9 bytes; common prefix of all keys: 33 bytes
  distinguishing prefix D = 8106449 bytes (40.53 per string)
                    bytes read    calls      moves
  MSD, no cutoff       8106449   232010   16212898   = D + E (E = 0): true
  MSD-16               8045714     8920   15526800   radix 7568425 + insertion 477289
  MSD-16 + skip        8045714     8920    2326800   same bytes, levels with one bucket not moved
  slices.SortFunc    134198320                       3706035 comparisons x 36.2 bytes
  bytes: slices.SortFunc / MSD-16 = 16.7x;  moves: MSD-16 / MSD-16 + skip = 6.7x

all sorts equal slices.Sort on both datasets: true

ns/element (copy-only control subtracted; best of 3)
dataset                      slices.Sort   MSD-16  MSD-16 + skip
(a) random 8-16 lowercase          125.8     34.5           26.9
(b) URLs                           124.1    190.2          106.8
```

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// A comparison sort sees a string only through "a < b?", and every answer
// starts at byte 0. Two URLs that share 33 bytes cost at least 34 byte
// compares EVERY time they meet, and they meet O(log n) times. MSD radix
// sort reads byte d of a string only while it still shares a bucket with
// another string -- so it reads each string's distinguishing prefix about
// once. This program counts both, then times them.

// ---- Go's math/rand/v2 PCG (PCG-DXSM), so the strings are Go's strings ----
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
}

/// An out-of-place MSD radix sort with an insertion-sort cutoff, over `&str`
/// (16 bytes each, like a Go string header). Bucket 0 is "string ended here";
/// byte b goes to bucket b+1, so "she" sorts before "shells". With `skip`, a
/// level where every string has the same byte is not distributed at all (the
/// pass skipping of example 13). The lifetime `'a` says: the buffer holds
/// borrowed strings and never outlives them.
struct Msd<'a> {
    cutoff: usize,
    skip: bool,
    aux: Vec<&'a str>,   // the n-element buffer
    dig: Vec<usize>,     // each string's digit at the current depth, read once
    reads: usize,        // bytes read by the radix passes
    insert_bytes: usize, // bytes compared inside insertion sort
    calls: usize,        // radix calls (each clears 258 counters)
    moves: usize,        // string headers written (to aux and back, swaps)
}

impl<'a> Msd<'a> {
    fn sort(&mut self, a: &mut [&'a str], d: usize) {
        if a.len() <= self.cutoff.max(1) {
            self.insertion(a, d);
            return;
        }
        self.calls += 1;
        let mut count = [0usize; 258];
        for (i, s) in a.iter().enumerate() {
            // Option replaces Go's `if d < len(s)`: no byte = end of string.
            let c = s.as_bytes().get(d).map_or(0, |&b| usize::from(b) + 1);
            self.dig[i] = c;
            count[c + 1] += 1;
            self.reads += 1;
        }
        let c0 = self.dig[0];
        if self.skip && count[c0 + 1] == a.len() {
            // one bucket holds all (c0 == 0: all ended here, so all are equal)
            if c0 > 0 {
                self.sort(a, d + 1);
            }
            return;
        }
        for r in 0..257 {
            // counts -> start positions
            count[r + 1] += count[r];
        }
        for (i, &s) in a.iter().enumerate() {
            // stable distribution into aux
            let c = self.dig[i];
            self.aux[count[c]] = s;
            count[c] += 1;
        }
        a.copy_from_slice(&self.aux[..a.len()]);
        self.moves += 2 * a.len();
        // Now count[c] is the END of bucket c. Bucket 0 (strings that ended) is
        // already in its final order: they are all equal. Recurse on the rest.
        for c in 1..257 {
            self.sort(&mut a[count[c - 1]..count[c]], d + 1);
        }
    }

    /// Sorts strings that all share their first d bytes, so it compares from
    /// byte d on and counts every byte it looks at.
    fn insertion(&mut self, a: &mut [&'a str], d: usize) {
        for i in 1..a.len() {
            let mut j = i;
            while j > 0 && self.less(a[j], a[j - 1], d) {
                a.swap(j, j - 1);
                self.moves += 2;
                j -= 1;
            }
        }
    }

    fn less(&mut self, s: &str, t: &str, d: usize) -> bool {
        // Every string in the bucket has at least d bytes (they share them).
        let (s, t) = (s.as_bytes(), t.as_bytes());
        for (x, y) in s[d..].iter().zip(&t[d..]) {
            self.insert_bytes += 1;
            if x != y {
                return x < y;
            }
        }
        s.len() < t.len()
    }
}

fn msd_sort<'a>(a: &mut [&'a str], cutoff: usize, skip: bool) -> Msd<'a> {
    let mut m = Msd {
        cutoff,
        skip,
        aux: vec![""; a.len()],
        dig: vec![0; a.len()],
        reads: 0,
        insert_bytes: 0,
        calls: 0,
        moves: 0,
    };
    m.sort(a, 0);
    m
}

/// Length of the longest common prefix of s and t.
fn lcp(s: &str, t: &str) -> usize {
    s.bytes().zip(t.bytes()).take_while(|(x, y)| x == y).count()
}

/// D, the total distinguishing prefix: for each string, one byte more than its
/// longest common prefix with any other string (capped at its length), and E,
/// how many strings are a prefix of another string or equal to one (MSD reads
/// one extra "end of string" digit for those).
fn distinguishing(sorted: &[&str]) -> (usize, usize) {
    let (mut d, mut e) = (0, 0);
    for (i, s) in sorted.iter().enumerate() {
        let mut l = 0;
        if i > 0 {
            l = lcp(sorted[i - 1], s);
        }
        if i + 1 < sorted.len() {
            l = l.max(lcp(s, sorted[i + 1]));
        }
        if l >= s.len() {
            d += s.len();
            e += 1;
        } else {
            d += l + 1;
        }
    }
    (d, e)
}

type Sorter = fn(&mut [&str]);

fn std_sort(a: &mut [&str]) {
    a.sort_unstable();
}
fn msd16(a: &mut [&str]) {
    black_box(msd_sort(a, 16, false).calls);
}
fn msd16_skip(a: &mut [&str]) {
    black_box(msd_sort(a, 16, true).calls);
}
fn copy_only(a: &mut [&str]) {
    black_box(a);
}

/// Copy the pristine order INSIDE the timed loop (a `&str` copy moves 16-byte
/// fat pointers, not the bytes), best of 3 batches of >= 20 ms.
fn ns_per_call<'a>(work: &mut [&'a str], pristine: &[&'a str], f: Sorter) -> f64 {
    let mut batch = |reps: u32| {
        let start = Instant::now();
        for _ in 0..reps {
            work.copy_from_slice(pristine);
            f(black_box(&mut *work));
        }
        start.elapsed()
    };
    let mut reps = 1;
    while batch(reps) < Duration::from_millis(20) {
        reps *= 2;
    }
    (0..3)
        .map(|_| batch(reps).as_nanos() as f64 / f64::from(reps))
        .fold(f64::INFINITY, f64::min)
}

fn main() {
    const N: usize = 200_000;
    const CUTOFF: usize = 16;
    let mut rng = Pcg::new(1, 2);

    // (a) random lowercase strings, 8..16 bytes. The owned Strings live here;
    // everything below sorts Vec<&str> views of them.
    let random: Vec<String> = (0..N)
        .map(|_| {
            let len = 8 + rng.next_u64() % 9;
            (0..len)
                .map(|_| char::from(b'a' + (rng.next_u64() % 26) as u8))
                .collect()
        })
        .collect();
    // (b) URLs sharing a 33-byte prefix.
    let urls: Vec<String> = (0..N)
        .map(|_| {
            let user = rng.next_u64() % 1_000_000; // drawn first, as in Go
            let order = rng.next_u64() % 100_000_000;
            format!("https://example.com/api/v1/users/{user:06}/orders/{order}")
        })
        .collect();

    // A fn, not a closure: a closure cannot say "the &strs borrow from v".
    fn views(v: &[String]) -> Vec<&str> {
        v.iter().map(String::as_str).collect()
    }
    let (random, urls) = (views(&random), views(&urls));
    let sets: [(&str, &[&str]); 2] = [("(a) random 8-16 lowercase", &random), ("(b) URLs", &urls)];

    let mut work: Vec<&str> = vec![""; N];
    let mut all_ok = true;
    println!("n = {N} strings per dataset, MSD cutoff = {CUTOFF}");
    for &(name, keys) in &sets {
        let mut reference = keys.to_vec();
        reference.sort_unstable();
        let total: usize = keys.iter().map(|k| k.len()).sum();
        let (d, e) = distinguishing(&reference);

        work.copy_from_slice(keys);
        let m0 = msd_sort(&mut work, 0, false);
        all_ok &= work == reference;
        work.copy_from_slice(keys);
        let mc = msd_sort(&mut work, CUTOFF, false);
        all_ok &= work == reference;
        work.copy_from_slice(keys);
        let ms = msd_sort(&mut work, CUTOFF, true);
        all_ok &= work == reference;
        let (mut cb, mut cc) = (0usize, 0usize);
        work.copy_from_slice(keys);
        // The counting comparator, as a closure that mutably borrows the two
        // counters for the duration of the sort.
        work.sort_unstable_by(|a, b| {
            cc += 1;
            for (x, y) in a.bytes().zip(b.bytes()) {
                cb += 1;
                if x != y {
                    return x.cmp(&y);
                }
            }
            a.len().cmp(&b.len())
        });
        all_ok &= work == reference;

        println!("\n{name}");
        println!(
            "  avg length {:.1} bytes; common prefix of all keys: {} bytes",
            total as f64 / N as f64,
            lcp(reference[0], reference[N - 1])
        );
        println!(
            "  distinguishing prefix D = {d} bytes ({:.2} per string)",
            d as f64 / N as f64
        );
        println!(
            "  {:<16} {:>11} {:>8} {:>10}",
            "", "bytes read", "calls", "moves"
        );
        let row = |name: &str, m: &Msd, note: String| {
            println!(
                "  {name:<16} {:>11} {:>8} {:>10}   {note}",
                m.reads + m.insert_bytes,
                m.calls,
                m.moves
            );
        };
        row(
            "MSD, no cutoff",
            &m0,
            format!("= D + E (E = {e}): {}", m0.reads == d + e),
        );
        row(
            "MSD-16",
            &mc,
            format!("radix {} + insertion {}", mc.reads, mc.insert_bytes),
        );
        row(
            "MSD-16 + skip",
            &ms,
            "same bytes, levels with one bucket not moved".to_string(),
        );
        println!(
            "  {:<16} {cb:>11} {:>8} {:>10}   {cc} comparisons x {:.1} bytes",
            "sort_unstable_by",
            "",
            "",
            cb as f64 / cc as f64
        );
        println!(
            "  bytes: sort_unstable_by / MSD-16 = {:.1}x;  moves: MSD-16 / MSD-16 + skip = {:.1}x",
            cb as f64 / (mc.reads + mc.insert_bytes) as f64,
            mc.moves as f64 / ms.moves as f64
        );
    }
    println!("\nall sorts equal sort_unstable on both datasets: {all_ok}");

    println!("\nns/element (copy-only control subtracted; best of 3)");
    println!(
        "{:<27} {:>14} {:>8} {:>14}",
        "dataset", "sort_unstable", "MSD-16", "MSD-16 + skip"
    );
    for &(name, keys) in &sets {
        let ctrl = ns_per_call(&mut work, keys, copy_only);
        let mut per = |f: Sorter| (ns_per_call(&mut work, keys, f) - ctrl) / N as f64;
        let (std, rad, skp) = (per(std_sort), per(msd16), per(msd16_skip));
        println!("{name:<27} {std:>14.1} {rad:>8.1} {skp:>14.1}");
    }
}
```

**Rust sample output:**

```
n = 200000 strings per dataset, MSD cutoff = 16

(a) random 8-16 lowercase
  avg length 12.0 bytes; common prefix of all keys: 0 bytes
  distinguishing prefix D = 873959 bytes (4.37 per string)
                    bytes read    calls      moves
  MSD, no cutoff        873959    52728    1747918   = D + E (E = 0): true
  MSD-16               1265249     1946    2194252   radix 622704 + insertion 642545
  MSD-16 + skip        1265249     1946    2194252   same bytes, levels with one bucket not moved
  sort_unstable_by     8733157                       3620778 comparisons x 2.4 bytes
  bytes: sort_unstable_by / MSD-16 = 6.9x;  moves: MSD-16 / MSD-16 + skip = 1.0x

(b) URLs
  avg length 54.9 bytes; common prefix of all keys: 33 bytes
  distinguishing prefix D = 8106449 bytes (40.53 per string)
                    bytes read    calls      moves
  MSD, no cutoff       8106449   232010   16212898   = D + E (E = 0): true
  MSD-16               8045714     8920   15526800   radix 7568425 + insertion 477289
  MSD-16 + skip        8045714     8920    2326800   same bytes, levels with one bucket not moved
  sort_unstable_by   131230410                       3621471 comparisons x 36.2 bytes
  bytes: sort_unstable_by / MSD-16 = 16.3x;  moves: MSD-16 / MSD-16 + skip = 6.7x

all sorts equal sort_unstable on both datasets: true

ns/element (copy-only control subtracted; best of 3)
dataset                      sort_unstable   MSD-16  MSD-16 + skip
(a) random 8-16 lowercase             34.6     23.6           25.2
(b) URLs                              64.9    232.2          119.0
```

> **Go vs Rust:** Every MSD count (D, E, bytes read, calls, moves) is identical to Go's: same PCG strings, same algorithm, with `Vec<&str>` views (16-byte fat pointers, like Go string headers) into owned `String`s, and `as_bytes().get(d)` returning `None` as the end-of-string digit. The comparator line differs because `sort_unstable_by` is ipnsort, not pdqsort: 3.62 million comparisons against Go's 3.71–3.76 million, at the same 36.2 bytes each on URLs.
> On the clock our MSD's ns/element are close to Go's (same code), but `sort_unstable` on `&str` is a much stronger opponent than `slices.Sort` (about 35 vs about 100 ns/element on (a); why was not measured). So across 7 runs MSD-16 won on random short strings by only 1.1–1.5x instead of Go's about 3.5x, and on URLs it lost 2–4x without skipping and, with skipping, came out anywhere from even to 2x slower, although it reads 16x fewer bytes.

**Complexity:** comparison sort Θ(n log n) comparisons, each costing up to the shared-prefix length (URLs: 36.2 bytes per comparison, 134 million bytes in all) · MSD Θ(D + R·calls) byte reads — without a cutoff exactly D + E (URLs: 8.1 million, and MSD-16 reads 16.7x fewer bytes than `slices.SortFunc`) · space Θ(n) buffer + digit cache + Θ(depth·R) counters · on the clock over 4 runs: MSD-16 was about 3.5x faster than `slices.Sort` on random short strings, but 1.4–1.5x slower on URLs despite reading 16.7x fewer bytes; skipping single-bucket levels cut its moves 6.7x and brought it to about even with `slices.Sort` (0.97–1.2x).

---

## 15. American flag sort: MSD radix in place

`🔴 hard` · *In-place radix*

MSD radix sort normally copies every bucket out to an n-element buffer and back. American flag sort
(McIlroy, Bostic & McIlroy, 1993) drops the buffer: it counts the bucket sizes, prefix-sums them into
bucket starts, and then **permutes in place** — pick up a key that is not home, drop it into its bucket's
next free slot, pick up what was there, and keep going until the cycle closes. Every swap puts one key in
its final bucket, so one level makes fewer than n swaps, and the extra memory is one 256-entry histogram
per recursion level. On 64-bit keys that is O(8n + 256·calls) time with Θ(depth·256) space and **no n
buffer** — and the price is stability, because a swap carries a key past its equal neighbours.

**Steps:**

1. Trace one permutation step on 12 32-bit keys by their top byte: the bucket boundaries, every carry, the result.
2. Sort records that share keys and compare the tag order with one stable counting pass on the same digit.
3. At n = 1,000,000 random `uint64`, count swaps, recursive calls and allocations, then time it against LSD-8 and `slices.Sort`.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"slices"
	"strings"
	"testing"
	"time"
)

// American flag sort (McIlroy, Bostic & McIlroy, 1993) is MSD radix sort
// without the n-element buffer. Per call, on one digit:
//
//  1. count how many keys fall in each of the 256 buckets
//  2. prefix-sum the counts into bucket starts
//  3. PERMUTE in place: walk each bucket's region; a key that belongs to
//     another bucket d is swapped into d's next free slot. Every swap puts one
//     key into its final bucket for good, so a pass makes fewer than n swaps.
//  4. recurse into each bucket on the next digit (insertion sort when small)
//
// Extra memory: one [256]int histogram per recursion level -- no n buffer.
// The price: a swap carries a key past its equal neighbours, so it is NOT stable.

const cutoff = 32 // buckets this small are finished by insertion sort

type stats struct {
	swaps      int // swaps made by the permutation steps
	passes     int // calls that counted and permuted (recursive calls)
	insertions int // buckets handed to insertion sort
}

func digit(x uint64, shift uint) int { return int(x >> shift & 0xFF) }

// permute does steps 1-3 for one digit and returns each bucket's size.
// trace, if not nil, is told about every key dropped into another bucket.
func permute(a []uint64, shift uint, st *stats, trace func(v uint64, j int, out uint64)) [256]int {
	var counts, next [256]int
	for _, x := range a {
		counts[digit(x, shift)]++
	}
	sum := 0
	for b := range 256 {
		next[b] = sum // next free slot of bucket b; starts at its start
		sum += counts[b]
	}
	end := 0
	for b := range 256 {
		end += counts[b] // bucket b is a[next[b]:end] until it is full
		for next[b] < end {
			// Pick up the first key not yet known to be home and carry it:
			// drop it into its bucket's next free slot, pick up whatever was
			// there, and repeat until we hold a key for bucket b. That is one
			// cycle of the permutation. Holding the key in v (instead of
			// swapping through a[next[b]]) halves the memory traffic.
			v := a[next[b]]
			d := digit(v, shift)
			for d != b {
				if trace != nil {
					trace(v, next[d], a[next[d]])
				}
				v, a[next[d]] = a[next[d]], v // the swap
				next[d]++
				st.swaps++
				d = digit(v, shift)
			}
			a[next[b]] = v // the cycle closes where it started
			next[b]++
		}
	}
	st.passes++
	return counts
}

// americanFlag sorts a by the digits at shift, shift-8, ..., 0.
func americanFlag(a []uint64, shift uint, st *stats) {
	counts := permute(a, shift, st, nil)
	if shift == 0 {
		return // last digit: every bucket holds equal keys
	}
	start := 0
	for _, c := range counts {
		bucket := a[start : start+c]
		start += c
		switch {
		case c > cutoff:
			americanFlag(bucket, shift-8, st)
		case c > 1:
			insertionSort(bucket)
			st.insertions++
		}
	}
}

func insertionSort(a []uint64) {
	for i := 1; i < len(a); i++ {
		for j := i; j > 0 && a[j] < a[j-1]; j-- {
			a[j], a[j-1] = a[j-1], a[j]
		}
	}
}

// lsd8 is the opponent from examples 5 and 12: 8 stable passes, one n buffer.
func lsd8(a []uint64) {
	buf := make([]uint64, len(a)) // the allocation American flag sort avoids
	src, dst := a, buf
	for shift := uint(0); shift < 64; shift += 8 {
		var counts [256]int
		for _, x := range src {
			counts[digit(x, shift)]++
		}
		sum := 0
		for b := range 256 {
			counts[b], sum = sum, sum+counts[b]
		}
		for _, x := range src {
			d := digit(x, shift)
			dst[counts[d]] = x
			counts[d]++
		}
		src, dst = dst, src
	}
	// 8 passes is even, so the result is back in a.
}

func hex32(a []uint64) string {
	parts := make([]string, len(a))
	for i, x := range a {
		parts[i] = fmt.Sprintf("%08x", x)
	}
	return strings.Join(parts, " ")
}

// nsPerElem times f (copy + sort) for at least 300 ms and 3 runs.
func nsPerElem(n int, f func()) float64 {
	runs, start := 0, time.Now()
	for runs < 3 || time.Since(start) < 300*time.Millisecond {
		f()
		runs++
	}
	return float64(time.Since(start).Nanoseconds()) / float64(runs) / float64(n)
}

// rel says how a time t compares with a reference time ref.
func rel(t, ref float64) string {
	if t <= ref {
		return fmt.Sprintf("%.1fx faster", ref/t)
	}
	return fmt.Sprintf("%.1fx slower", t/ref)
}

func bytesAllocated(f func()) uint64 {
	var m0, m1 runtime.MemStats
	runtime.ReadMemStats(&m0)
	f()
	runtime.ReadMemStats(&m1)
	return m1.TotalAlloc - m0.TotalAlloc
}

func main() {
	// ---- 1. one permutation step, traced -------------------------------
	fmt.Println("1. 12 uint32 keys, top byte only (shift 24):")
	a := []uint64{
		0x3a01c2f0, 0x07ff0001, 0xc0de0042, 0x3a00beef, 0x07000a0b, 0x91a2b3c4,
		0xc0010203, 0x3affffff, 0x91000000, 0x07123456, 0x3a5a5a5a, 0xc0ffee00,
	}
	fmt.Println("   input:  ", hex32(a))
	sortedA := slices.Clone(a)
	slices.Sort(sortedA)
	var st stats
	counts := permute(slices.Clone(a), 24, &st, nil) // dry run for the boundaries
	fmt.Print("   buckets:")
	start := 0
	for b, c := range counts {
		if c > 0 {
			fmt.Printf(" %02x:[%d,%d)", b, start, start+c)
			start += c
		}
	}
	fmt.Println()
	st = stats{}
	permute(a, 24, &st, func(v uint64, j int, out uint64) {
		fmt.Printf("   carry %08x -> a[%2d] (bucket %02x), pick up %08x\n", v, j, digit(v, 24), out)
	})
	fmt.Println("   after:  ", hex32(a))
	fmt.Printf("   %d swaps for %d keys: each bucket is now contiguous, its inside not yet sorted\n", st.swaps, len(a))
	for i := 0; i < len(a); {
		j := i
		for j < len(a) && digit(a[j], 24) == digit(a[i], 24) {
			j++
		}
		insertionSort(a[i:j]) // the recursion would go to shift 16; small, so insertion sort
		i = j
	}
	fmt.Println("   sorted: ", hex32(a))
	fmt.Println("   equals slices.Sort:", slices.Equal(a, sortedA))

	// ---- 2. stability ----------------------------------------------------
	// Record = key in the top byte, tag (a letter, in input order) in the low
	// byte. Sorting on the top byte alone sorts by key and carries the tag.
	fmt.Println()
	fmt.Println("2. 12 records (key, tag), tags in input order; sort by key only:")
	keys := []uint64{2, 1, 3, 2, 1, 1, 3, 2, 3, 1, 2, 3}
	recs := make([]uint64, len(keys))
	for i, k := range keys {
		recs[i] = k<<56 | uint64('a'+i)
	}
	show := func(rs []uint64) string {
		var sb strings.Builder
		for _, r := range rs {
			fmt.Fprintf(&sb, " %d%c", r>>56, byte(r))
		}
		return sb.String()
	}
	stable := func(rs []uint64) bool { // within equal keys, tags increase
		for i := 1; i < len(rs); i++ {
			if rs[i]>>56 == rs[i-1]>>56 && byte(rs[i]) < byte(rs[i-1]) {
				return false
			}
		}
		return true
	}
	fmt.Println("   input:          ", show(recs))
	af := slices.Clone(recs)
	permute(af, 56, &stats{}, nil)
	fmt.Println("   American flag:  ", show(af), "  stable:", stable(af))
	ls := slices.Clone(recs) // one stable counting pass on the same digit (LSD)
	var c [256]int
	for _, r := range ls {
		c[r>>56]++
	}
	for b, sum := 0, 0; b < 256; b++ {
		c[b], sum = sum, sum+c[b]
	}
	out := make([]uint64, len(ls))
	for _, r := range ls {
		out[c[r>>56]] = r
		c[r>>56]++
	}
	fmt.Println("   counting pass:  ", show(out), "  stable:", stable(out))

	// ---- 3. counts at n = 1M ----------------------------------------------
	const n = 1_000_000
	rng := rand.New(rand.NewPCG(1, 2))
	pristine := make([]uint64, 4*n)
	for i := range pristine {
		pristine[i] = rng.Uint64()
	}
	in1 := pristine[:n]
	want := slices.Clone(in1)
	slices.Sort(want)

	fmt.Println()
	fmt.Printf("3. n = %d random uint64 (PCG 1,2), cutoff %d:\n", n, cutoff)
	work := slices.Clone(in1)
	st = stats{}
	americanFlag(work, 56, &st)
	fmt.Println("   equals slices.Sort:", slices.Equal(work, want))
	fmt.Printf("   swaps:                    %d  (%.3f per key)\n", st.swaps, float64(st.swaps)/n)
	fmt.Printf("   recursive calls (passes): %d\n", st.passes)
	fmt.Printf("   buckets insertion-sorted: %d\n", st.insertions)
	lsdOut := slices.Clone(in1)
	lsd8(lsdOut)
	fmt.Println("   LSD-8 equals slices.Sort:", slices.Equal(lsdOut, want))

	work = make([]uint64, n)
	afAllocs := testing.AllocsPerRun(3, func() { copy(work, in1); americanFlag(work, 56, &stats{}) })
	lsdAllocs := testing.AllocsPerRun(3, func() { copy(work, in1); lsd8(work) })
	afBytes := bytesAllocated(func() { copy(work, in1); americanFlag(work, 56, &stats{}) })
	lsdBytes := bytesAllocated(func() { copy(work, in1); lsd8(work) })
	fmt.Printf("   %-15s %12s %14s\n", "per call", "allocations", "bytes")
	fmt.Printf("   %-15s %12.0f %14d\n", "American flag", afAllocs, afBytes)
	note := ""
	if pages := (8*n + 8191) / 8192 * 8192; lsdBytes == uint64(pages) {
		note = fmt.Sprintf("  = 8n (%d) rounded up to whole 8 KiB pages", 8*n)
	}
	fmt.Printf("   %-15s %12.0f %14d%s\n", "LSD-8", lsdAllocs, lsdBytes, note)

	// ---- 4. timing ------------------------------------------------------------
	fmt.Println()
	fmt.Println("4. ns per element; every run copies a pristine input first (copy-only row = that cost):")
	fmt.Printf("   %-15s %10s %10s\n", "", "n = 1M", "n = 4M")
	type row struct {
		name string
		sort func([]uint64)
	}
	rows := []row{
		{"copy only", func([]uint64) {}},
		{"American flag", func(x []uint64) { americanFlag(x, 56, &stats{}) }},
		{"LSD-8", lsd8},
		{"slices.Sort", slices.Sort[[]uint64]},
	}
	// correctness at 4M before timing anything there
	want4 := slices.Clone(pristine)
	slices.Sort(want4)
	for _, r := range rows[1:] {
		x := slices.Clone(pristine)
		r.sort(x)
		if !slices.Equal(x, want4) {
			panic(r.name + " is wrong at 4M")
		}
	}
	res := map[string][2]float64{}
	for _, r := range rows {
		var cell [2]float64
		for k, m := range []int{n, 4 * n} {
			in := pristine[:m]
			w := make([]uint64, m)
			cell[k] = nsPerElem(m, func() { copy(w, in); r.sort(w) })
		}
		res[r.name] = cell
		fmt.Printf("   %-15s %10.1f %10.1f\n", r.name, cell[0], cell[1])
	}
	for k, label := range []string{"1M", "4M"} {
		f, l, s := res["American flag"][k], res["LSD-8"][k], res["slices.Sort"][k]
		fmt.Printf("   at %s: American flag is %s than slices.Sort and %s than LSD-8\n", label, rel(f, s), rel(f, l))
	}
}
```

**Sample output:**

```
1. 12 uint32 keys, top byte only (shift 24):
   input:   3a01c2f0 07ff0001 c0de0042 3a00beef 07000a0b 91a2b3c4 c0010203 3affffff 91000000 07123456 3a5a5a5a c0ffee00
   buckets: 07:[0,3) 3a:[3,7) 91:[7,9) c0:[9,12)
   carry 3a01c2f0 -> a[ 3] (bucket 3a), pick up 3a00beef
   carry 3a00beef -> a[ 4] (bucket 3a), pick up 07000a0b
   carry c0de0042 -> a[ 9] (bucket c0), pick up 07123456
   carry 91a2b3c4 -> a[ 7] (bucket 91), pick up 3affffff
   carry c0010203 -> a[10] (bucket c0), pick up 3a5a5a5a
   after:   07000a0b 07ff0001 07123456 3a01c2f0 3a00beef 3affffff 3a5a5a5a 91a2b3c4 91000000 c0de0042 c0010203 c0ffee00
   5 swaps for 12 keys: each bucket is now contiguous, its inside not yet sorted
   sorted:  07000a0b 07123456 07ff0001 3a00beef 3a01c2f0 3a5a5a5a 3affffff 91000000 91a2b3c4 c0010203 c0de0042 c0ffee00
   equals slices.Sort: true

2. 12 records (key, tag), tags in input order; sort by key only:
   input:            2a 1b 3c 2d 1e 1f 3g 2h 3i 1j 2k 3l
   American flag:    1e 1b 1j 1f 2a 2d 2k 2h 3c 3i 3g 3l   stable: false
   counting pass:    1b 1e 1f 1j 2a 2d 2h 2k 3c 3g 3i 3l   stable: true

3. n = 1000000 random uint64 (PCG 1,2), cutoff 32:
   equals slices.Sort: true
   swaps:                    1990574  (1.991 per key)
   recursive calls (passes): 260
   buckets insertion-sorted: 65543
   LSD-8 equals slices.Sort: true
   per call         allocations          bytes
   American flag              0              0
   LSD-8                      1        8003584  = 8n (8000000) rounded up to whole 8 KiB pages

4. ns per element; every run copies a pristine input first (copy-only row = that cost):
                       n = 1M     n = 4M
   copy only              0.1        0.2
   American flag         18.5       23.0
   LSD-8                  6.9        7.0
   slices.Sort           58.2       64.5
   at 1M: American flag is 3.1x faster than slices.Sort and 2.7x slower than LSD-8
   at 4M: American flag is 2.8x faster than slices.Sort and 3.3x slower than LSD-8
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// American flag sort (McIlroy, Bostic & McIlroy, 1993) is MSD radix sort
// without the n-element buffer. Per call, on one digit:
//
//  1. count how many keys fall in each of the 256 buckets
//  2. prefix-sum the counts into bucket starts
//  3. PERMUTE in place: walk each bucket's region; a key that belongs to
//     another bucket d is swapped into d's next free slot. Every swap puts one
//     key into its final bucket for good, so a pass makes fewer than n swaps.
//  4. recurse into each bucket on the next digit (insertion sort when small)
//
// Extra memory: one [usize; 256] histogram per recursion level -- no n buffer.
// The price: a swap carries a key past its equal neighbours, so it is NOT stable.

// ---- allocation counter (stand-in for testing.AllocsPerRun / MemStats) ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static TOTAL: AtomicUsize = AtomicUsize::new(0);

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
fn allocs_of(f: impl FnOnce()) -> (usize, usize) {
    let (a0, t0) = (ALLOCS.load(Relaxed), TOTAL.load(Relaxed));
    f();
    (ALLOCS.load(Relaxed) - a0, TOTAL.load(Relaxed) - t0)
}

// ---- Go's math/rand/v2 PCG (PCG-DXSM), so the keys are the same as Go's ----
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
}

const CUTOFF: usize = 32; // buckets this small are finished by insertion sort

#[derive(Default)]
struct Stats {
    swaps: usize,      // swaps made by the permutation steps
    passes: usize,     // calls that counted and permuted (recursive calls)
    insertions: usize, // buckets handed to insertion sort
}

fn digit(x: u64, shift: u32) -> usize {
    ((x >> shift) & 0xFF) as usize
}

// The trace hook gets (key carried, slot it lands in, key picked up there).
// It cannot read `a` itself: `permute` holds the only &mut to it.
type Trace<'t> = Option<&'t mut dyn FnMut(u64, usize, u64)>;

/// Steps 1-3 for one digit; returns each bucket's size.
fn permute(a: &mut [u64], shift: u32, st: &mut Stats, mut trace: Trace) -> [usize; 256] {
    let mut counts = [0usize; 256];
    for &x in a.iter() {
        counts[digit(x, shift)] += 1;
    }
    let mut next = [0usize; 256]; // next free slot of bucket b; starts at its start
    let mut sum = 0;
    for (nx, &c) in next.iter_mut().zip(&counts) {
        *nx = sum;
        sum += c;
    }
    let mut end = 0;
    for b in 0..256 {
        end += counts[b]; // bucket b is a[next[b]..end] until it is full
        while next[b] < end {
            // Pick up the first key not yet known to be home and carry it:
            // drop it into its bucket's next free slot, pick up whatever was
            // there, and repeat until we hold a key for bucket b. That is one
            // cycle of the permutation. Holding the key in v (instead of
            // swapping through a[next[b]]) halves the memory traffic.
            let mut v = a[next[b]];
            let mut d = digit(v, shift);
            while d != b {
                let j = next[d];
                if let Some(t) = trace.as_mut() {
                    t(v, j, a[j]);
                }
                std::mem::swap(&mut v, &mut a[j]); // the swap
                next[d] += 1;
                st.swaps += 1;
                d = digit(v, shift);
            }
            a[next[b]] = v; // the cycle closes where it started
            next[b] += 1;
        }
    }
    st.passes += 1;
    counts
}

/// Sorts `a` by the digits at shift, shift-8, ..., 0.
fn american_flag(a: &mut [u64], shift: u32, st: &mut Stats) {
    let counts = permute(a, shift, st, None);
    if shift == 0 {
        return; // last digit: every bucket holds equal keys
    }
    // Hand each bucket to the recursion as its own &mut subslice. split_at_mut
    // proves to the borrow checker that the pieces are disjoint; mem::take
    // moves `rest` out so the tail can be reborrowed for the next round.
    let mut rest: &mut [u64] = a;
    for &c in &counts {
        let (bucket, tail) = std::mem::take(&mut rest).split_at_mut(c);
        rest = tail;
        if c > CUTOFF {
            american_flag(bucket, shift - 8, st);
        } else if c > 1 {
            insertion_sort(bucket);
            st.insertions += 1;
        }
    }
}

fn insertion_sort(a: &mut [u64]) {
    for i in 1..a.len() {
        let mut j = i;
        while j > 0 && a[j] < a[j - 1] {
            a.swap(j, j - 1);
            j -= 1;
        }
    }
}

/// The opponent from examples 5 and 12: 8 stable passes, one n buffer.
fn lsd8(a: &mut [u64]) {
    let mut buf = vec![0u64; a.len()]; // the allocation American flag sort avoids
    let (mut src, mut dst): (&mut [u64], &mut [u64]) = (a, &mut buf);
    for shift in (0..64).step_by(8) {
        let mut counts = [0usize; 256];
        for &x in src.iter() {
            counts[digit(x, shift)] += 1;
        }
        let mut sum = 0;
        for c in counts.iter_mut() {
            (*c, sum) = (sum, sum + *c);
        }
        for &x in src.iter() {
            let d = digit(x, shift);
            dst[counts[d]] = x;
            counts[d] += 1;
        }
        std::mem::swap(&mut src, &mut dst);
    }
    // 8 passes is even, so the result is back in a.
}

fn hex32(a: &[u64]) -> String {
    a.iter()
        .map(|x| format!("{x:08x}"))
        .collect::<Vec<_>>()
        .join(" ")
}

type SortFn = fn(&mut [u64]);

/// Times f (copy + sort) for at least 300 ms and 3 runs.
fn ns_per_elem(n: usize, mut f: impl FnMut()) -> f64 {
    let (mut runs, start) = (0u32, Instant::now());
    while runs < 3 || start.elapsed() < Duration::from_millis(300) {
        f();
        runs += 1;
    }
    start.elapsed().as_nanos() as f64 / f64::from(runs) / n as f64
}

/// How a time t compares with a reference time.
fn rel(t: f64, reference: f64) -> String {
    if t <= reference {
        format!("{:.1}x faster", reference / t)
    } else {
        format!("{:.1}x slower", t / reference)
    }
}

fn main() {
    // ---- 1. one permutation step, traced -------------------------------
    println!("1. 12 uint32 keys, top byte only (shift 24):");
    let mut a: Vec<u64> = vec![
        0x3a01c2f0, 0x07ff0001, 0xc0de0042, 0x3a00beef, 0x07000a0b, 0x91a2b3c4, 0xc0010203,
        0x3affffff, 0x91000000, 0x07123456, 0x3a5a5a5a, 0xc0ffee00,
    ];
    println!("   input:   {}", hex32(&a));
    let mut sorted_a = a.clone();
    sorted_a.sort_unstable();
    let counts = permute(&mut a.clone(), 24, &mut Stats::default(), None); // dry run
    print!("   buckets:");
    let mut start = 0;
    for (b, &c) in counts.iter().enumerate().filter(|&(_, &c)| c > 0) {
        print!(" {b:02x}:[{start},{})", start + c);
        start += c;
    }
    println!();
    let mut st = Stats::default();
    let mut show = |v: u64, j: usize, out: u64| {
        println!(
            "   carry {v:08x} -> a[{j:2}] (bucket {:02x}), pick up {out:08x}",
            digit(v, 24)
        );
    };
    permute(&mut a, 24, &mut st, Some(&mut show));
    println!("   after:   {}", hex32(&a));
    println!(
        "   {} swaps for {} keys: each bucket is now contiguous, its inside not yet sorted",
        st.swaps,
        a.len()
    );
    // the recursion would go to shift 16; the buckets are small, so insertion sort
    for bucket in a.chunk_by_mut(|x, y| digit(*x, 24) == digit(*y, 24)) {
        insertion_sort(bucket);
    }
    println!("   sorted:  {}", hex32(&a));
    println!("   equals sort_unstable: {}", a == sorted_a);

    // ---- 2. stability ----------------------------------------------------
    // Record = key in the top byte, tag (a letter, in input order) in the low
    // byte. Sorting on the top byte alone sorts by key and carries the tag.
    println!();
    println!("2. 12 records (key, tag), tags in input order; sort by key only:");
    let keys: [u64; 12] = [2, 1, 3, 2, 1, 1, 3, 2, 3, 1, 2, 3];
    let recs: Vec<u64> = (0..)
        .zip(keys)
        .map(|(i, k)| k << 56 | (b'a' + i) as u64)
        .collect();
    let show = |rs: &[u64]| -> String {
        rs.iter()
            .map(|&r| format!(" {}{}", r >> 56, r as u8 as char))
            .collect()
    };
    // within equal keys, tags increase
    let stable = |rs: &[u64]| {
        rs.windows(2)
            .all(|w| w[0] >> 56 != w[1] >> 56 || w[0] as u8 <= w[1] as u8)
    };
    println!("   input:           {}", show(&recs));
    let mut af = recs.clone();
    permute(&mut af, 56, &mut Stats::default(), None);
    println!(
        "   American flag:   {}   stable: {}",
        show(&af),
        stable(&af)
    );
    let mut c = [0usize; 256]; // one stable counting pass on the same digit (LSD)
    for &r in &recs {
        c[(r >> 56) as usize] += 1;
    }
    let mut sum = 0;
    for x in c.iter_mut() {
        (*x, sum) = (sum, sum + *x);
    }
    let mut out = vec![0u64; recs.len()];
    for &r in &recs {
        out[c[(r >> 56) as usize]] = r;
        c[(r >> 56) as usize] += 1;
    }
    println!(
        "   counting pass:   {}   stable: {}",
        show(&out),
        stable(&out)
    );

    // ---- 3. counts at n = 1M ----------------------------------------------
    const N: usize = 1_000_000;
    let mut rng = Pcg::new(1, 2);
    let pristine: Vec<u64> = (0..4 * N).map(|_| rng.next_u64()).collect();
    let in1 = &pristine[..N];
    let mut want = in1.to_vec();
    want.sort_unstable();

    println!();
    println!("3. n = {N} random uint64 (PCG 1,2), cutoff {CUTOFF}:");
    let mut work = in1.to_vec();
    let mut st = Stats::default();
    american_flag(&mut work, 56, &mut st);
    println!("   equals sort_unstable: {}", work == want);
    println!(
        "   swaps:                    {}  ({:.3} per key)",
        st.swaps,
        st.swaps as f64 / N as f64
    );
    println!("   recursive calls (passes): {}", st.passes);
    println!("   buckets insertion-sorted: {}", st.insertions);
    let mut lsd_out = in1.to_vec();
    lsd8(&mut lsd_out);
    println!("   LSD-8 equals sort_unstable: {}", lsd_out == want);

    let (af_allocs, af_bytes) = allocs_of(|| {
        work.copy_from_slice(in1);
        american_flag(&mut work, 56, &mut Stats::default());
    });
    let (lsd_allocs, lsd_bytes) = allocs_of(|| {
        work.copy_from_slice(in1);
        lsd8(&mut work);
    });
    let (su_allocs, su_bytes) = allocs_of(|| {
        work.copy_from_slice(in1);
        work.sort_unstable();
    });
    println!(
        "   {:<15} {:>12} {:>14}",
        "per call", "allocations", "bytes"
    );
    println!("   {:<15} {af_allocs:>12} {af_bytes:>14}", "American flag");
    let note = if lsd_bytes == 8 * N {
        format!("  = 8n exactly ({})", 8 * N)
    } else {
        String::new()
    };
    println!("   {:<15} {lsd_allocs:>12} {lsd_bytes:>14}{note}", "LSD-8");
    println!("   {:<15} {su_allocs:>12} {su_bytes:>14}", "sort_unstable");

    // ---- 4. timing ------------------------------------------------------------
    println!();
    println!(
        "4. ns per element; every run copies a pristine input first (copy-only row = that cost):"
    );
    println!("   {:<15} {:>10} {:>10}", "", "n = 1M", "n = 4M");
    let rows: [(&str, SortFn); 4] = [
        ("copy only", |_| {}),
        ("American flag", |x| {
            american_flag(x, 56, &mut Stats::default())
        }),
        ("LSD-8", lsd8),
        ("sort_unstable", |x| x.sort_unstable()),
    ];
    // correctness at 4M before timing anything there
    let mut want4 = pristine.clone();
    want4.sort_unstable();
    for (name, sort) in &rows[1..] {
        let mut x = pristine.clone();
        sort(&mut x);
        assert!(x == want4, "{name} is wrong at 4M");
    }
    let mut res = Vec::new();
    for (name, sort) in &rows {
        let mut cell = [0.0; 2];
        for (k, m) in [N, 4 * N].into_iter().enumerate() {
            let input = &pristine[..m];
            let mut w = vec![0u64; m];
            cell[k] = ns_per_elem(m, || {
                w.copy_from_slice(input);
                sort(black_box(&mut w));
            });
        }
        println!("   {name:<15} {:>10.1} {:>10.1}", cell[0], cell[1]);
        res.push(cell);
    }
    for (k, label) in ["1M", "4M"].into_iter().enumerate() {
        let (f, l, s) = (res[1][k], res[2][k], res[3][k]);
        println!(
            "   at {label}: American flag is {} than sort_unstable and {} than LSD-8",
            rel(f, s),
            rel(f, l)
        );
    }
}
```

**Rust sample output:**

```
1. 12 uint32 keys, top byte only (shift 24):
   input:   3a01c2f0 07ff0001 c0de0042 3a00beef 07000a0b 91a2b3c4 c0010203 3affffff 91000000 07123456 3a5a5a5a c0ffee00
   buckets: 07:[0,3) 3a:[3,7) 91:[7,9) c0:[9,12)
   carry 3a01c2f0 -> a[ 3] (bucket 3a), pick up 3a00beef
   carry 3a00beef -> a[ 4] (bucket 3a), pick up 07000a0b
   carry c0de0042 -> a[ 9] (bucket c0), pick up 07123456
   carry 91a2b3c4 -> a[ 7] (bucket 91), pick up 3affffff
   carry c0010203 -> a[10] (bucket c0), pick up 3a5a5a5a
   after:   07000a0b 07ff0001 07123456 3a01c2f0 3a00beef 3affffff 3a5a5a5a 91a2b3c4 91000000 c0de0042 c0010203 c0ffee00
   5 swaps for 12 keys: each bucket is now contiguous, its inside not yet sorted
   sorted:  07000a0b 07123456 07ff0001 3a00beef 3a01c2f0 3a5a5a5a 3affffff 91000000 91a2b3c4 c0010203 c0de0042 c0ffee00
   equals sort_unstable: true

2. 12 records (key, tag), tags in input order; sort by key only:
   input:            2a 1b 3c 2d 1e 1f 3g 2h 3i 1j 2k 3l
   American flag:    1e 1b 1j 1f 2a 2d 2k 2h 3c 3i 3g 3l   stable: false
   counting pass:    1b 1e 1f 1j 2a 2d 2h 2k 3c 3g 3i 3l   stable: true

3. n = 1000000 random uint64 (PCG 1,2), cutoff 32:
   equals sort_unstable: true
   swaps:                    1990574  (1.991 per key)
   recursive calls (passes): 260
   buckets insertion-sorted: 65543
   LSD-8 equals sort_unstable: true
   per call         allocations          bytes
   American flag              0              0
   LSD-8                      1        8000000  = 8n exactly (8000000)
   sort_unstable              0              0

4. ns per element; every run copies a pristine input first (copy-only row = that cost):
                       n = 1M     n = 4M
   copy only              0.2        0.5
   American flag         16.8       17.9
   LSD-8                  6.8        6.7
   sort_unstable         10.2       11.2
   at 1M: American flag is 1.6x slower than sort_unstable and 2.5x slower than LSD-8
   at 4M: American flag is 1.6x slower than sort_unstable and 2.7x slower than LSD-8
```

> **Go vs Rust:** Sections 1–3 match Go line for line (same PCG keys: 1,990,574 swaps, 260 passes, 65,543 insertion-sorted buckets), except the stdlib label and LSD-8's bytes: Rust's counter sees the 8,000,000 bytes requested, Go's `TotalAlloc` reports 8,003,584 because a large Go allocation is rounded up to whole 8 KiB pages. The recursion hands each bucket over with `split_at_mut`, which the borrow checker accepts only because the buckets are disjoint, and the trace closure gets the keys as arguments because `permute` holds the only `&mut` to the array.
> The timing verdict flips: American flag (~13–22 ns/element over several runs) beats Go's `slices.Sort` about 3x, but loses 1.1–1.9x to Rust's `sort_unstable` (ipnsort, ~10–13 ns/element here); LSD-8 runs ~7 ns/element in both languages and stays the fastest.

**Complexity:** time O(8n + 256·calls) on 64-bit keys, the cutoff of 32 bounding `calls` (260 counting passes, about 2 swaps per key at n = 1M) · space Θ(depth·256) — 0 allocations against LSD-8's one 8 MB buffer · stable: no. Measured over several runs: American flag ~18–24 ns/element, LSD-8 ~7–9, `slices.Sort` ~58–67 — in place costs 2–3x LSD's buffered passes and still beats pdqsort about 3x.

---

## 16. Capstone: `radix`, built, proved, measured

`🔴 hard` · *Capstone*

The whole lesson as one small library, treated like production code. `SortUint64`, `SortInt64`,
`SortFloat64`, `SortStrings` and a stable `SortByKey` share one design: map the key to `uint64` without
changing its order (example 8), read the input **once** for min, max and all eight digit histograms, take
the counting path when the range is at most n (example 4), otherwise run LSD with one reusable buffer and
**skip every pass whose digit is constant** (example 13); strings go to MSD with an insertion-sort cutoff
(example 9); and below n = 1,024 — the crossover example 12 measured — everything defers to the stdlib.
That is Θ(p·n + p·2⁸) for the p passes actually needed, Θ(n + k) on the counting path. A library is only
as good as its proof, so before any timing each function meets an oracle on 20,000 random cases built
to produce ties, plus the edge cases that break radix sorts: extremes, ±0, NaNs, empty strings, prefixes.

**Steps:**

1. Build the five functions on one core: key transform, one-read histograms, counting path, pass-skipping LSD, MSD, stdlib fallback.
2. Prove them with the threshold forced to 0 (so our code runs on every case), then through the public API, against `slices.Sort`, a hand-written total-order comparator and `slices.SortStableFunc`.
3. Time five workloads at n = 1,000,000 against `slices.Sort` and read the path each one took.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"strings"
	"testing"
	"time"
)

// ===================== package radix (one file here) =====================
//
// SortUint64, SortInt64, SortFloat64, SortStrings and the stable SortByKey,
// assembled from the lesson: order-preserving key transforms (ex 8), one read
// for all eight histograms and pass skipping (ex 13), counting sort when the
// range is at most n (ex 4), MSD with a cutoff for strings (ex 9), one buffer
// per call that every pass reuses, and the stdlib below a threshold (ex 12).

// smallN: below this, defer to the stdlib. Example 12 measured lsd8 first
// beating slices.Sort at n = 1K on this machine; we take that as the constant.
const smallN = 1024

const strCutoff = 32 // MSD buckets this small are finished by insertion sort

type path int

const (
	stdPath path = iota
	countPath
	radixPath
)

// info says what a call did (the prover and the benchmark print it).
type info struct {
	path   path
	passes int    // LSD byte passes actually run, of 8
	k      uint64 // counting sort: the range max-min+1
}

func SortUint64(xs []uint64)   { sortInts(xs, 0, smallN) }
func SortInt64(xs []int64)     { sortInts(xs, 1<<63, smallN) }
func SortFloat64(xs []float64) { sortFloats(xs, smallN) }
func SortStrings(xs []string)  { sortStrings(xs, smallN) }

// SortByKey sorts stably by key(x), ascending.
func SortByKey[T any](xs []T, key func(T) uint64) { sortByKey(xs, key, smallN) }

// sortInts: key = bits ^ flip; flip = sign bit for int64 (ex 8), 0 for uint64.
func sortInts[T ~uint64 | ~int64](xs []T, flip uint64, small int) info {
	n := len(xs)
	if n < max(small, 2) {
		slices.Sort(xs)
		return info{path: stdPath}
	}
	var hist [8][256]int // one read: min, max and every digit's histogram
	lo, hi := ^uint64(0), uint64(0)
	for _, x := range xs {
		k := uint64(x) ^ flip
		lo, hi = min(lo, k), max(hi, k)
		for p := range 8 {
			hist[p][byte(k>>(8*p))]++
		}
	}
	if hi-lo < uint64(n) { // range <= n: counting sort, rewrite from counts
		counts := make([]int, hi-lo+1)
		for _, x := range xs {
			counts[uint64(x)^flip-lo]++
		}
		i := 0
		for v, c := range counts {
			for x := T((lo + uint64(v)) ^ flip); c > 0; c-- {
				xs[i] = x
				i++
			}
		}
		return info{path: countPath, k: hi - lo + 1}
	}
	buf := make([]T, n) // the one buffer, reused by every pass
	src, dst, run := xs, buf, 0
	for p := range 8 {
		shift := 8 * p
		if hist[p][byte((uint64(src[0])^flip)>>shift)] == n {
			continue // every key has the same digit here: skip the pass
		}
		var pos [256]int
		for b, sum := 0, 0; b < 256; b++ {
			pos[b], sum = sum, sum+hist[p][b]
		}
		for _, x := range src {
			d := byte((uint64(x) ^ flip) >> shift)
			dst[pos[d]] = x
			pos[d]++
		}
		src, dst, run = dst, src, run+1
	}
	if run%2 == 1 {
		copy(xs, src) // an odd number of passes ends in buf
	}
	return info{path: radixPath, passes: run}
}

// floatKey is the IEEE totalOrder transform: negatives flip every bit,
// non-negatives flip the sign bit. keyFloat undoes it, bit for bit.
func floatKey(f float64) uint64 {
	b := math.Float64bits(f)
	return b ^ (uint64(int64(b)>>63) | 1<<63)
}
func keyFloat(k uint64) float64 { return math.Float64frombits(k ^ (uint64(int64(^k)>>63) | 1<<63)) }

func sortFloats(xs []float64, small int) info {
	if len(xs) < max(small, 2) {
		// NOT slices.Sort: it puts NaNs first and treats -0 == +0, so the
		// order would change with n. The fallback must use the same order.
		slices.SortFunc(xs, func(a, b float64) int { return cmp.Compare(floatKey(a), floatKey(b)) })
		return info{path: stdPath}
	}
	// Go cannot view a []float64 as []uint64 without unsafe, so the keys get
	// their own slice: a second n-sized allocation.
	keys := make([]uint64, len(xs))
	for i, f := range xs {
		keys[i] = floatKey(f)
	}
	in := sortInts(keys, 0, 0)
	for i, k := range keys {
		xs[i] = keyFloat(k)
	}
	return in
}

func sortStrings(xs []string, small int) info {
	if len(xs) < max(small, 2) {
		slices.Sort(xs)
		return info{path: stdPath}
	}
	msd(xs, make([]string, len(xs)), 0)
	return info{path: radixPath}
}

// charAt: 0 = end of string (below every byte), else byte+1.
func charAt(s string, d int) int {
	if d < len(s) {
		return int(s[d]) + 1
	}
	return 0
}

// msd sorts xs, whose strings all share their first d bytes.
func msd(xs, buf []string, d int) {
	for {
		if len(xs) <= strCutoff {
			for i := 1; i < len(xs); i++ { // insertion sort on the suffixes
				for j := i; j > 0 && xs[j][d:] < xs[j-1][d:]; j-- {
					xs[j], xs[j-1] = xs[j-1], xs[j]
				}
			}
			return
		}
		var counts [257]int
		for _, s := range xs {
			counts[charAt(s, d)]++
		}
		if c := charAt(xs[0], d); counts[c] == len(xs) { // one bucket holds all
			if c == 0 {
				return // all strings ended: all equal
			}
			d++ // a shared byte: skip it without moving anything
			continue
		}
		var pos [257]int
		for c, sum := 0, 0; c < 257; c++ {
			pos[c], sum = sum, sum+counts[c]
		}
		for _, s := range xs {
			c := charAt(s, d)
			buf[pos[c]] = s
			pos[c]++
		}
		copy(xs, buf[:len(xs)])
		for c, start := 1, counts[0]; c < 257; c++ { // bucket 0 is done
			if counts[c] > 1 {
				msd(xs[start:start+counts[c]], buf[start:start+counts[c]], d+1)
			}
			start += counts[c]
		}
		return
	}
}

func sortByKey[T any](xs []T, key func(T) uint64, small int) info {
	n := len(xs)
	if n < max(small, 2) {
		slices.SortStableFunc(xs, func(a, b T) int { return cmp.Compare(key(a), key(b)) })
		return info{path: stdPath}
	}
	var hist [8][256]int
	for _, x := range xs {
		k := key(x)
		for p := range 8 {
			hist[p][byte(k>>(8*p))]++
		}
	}
	buf := make([]T, n)
	src, dst, run := xs, buf, 0
	for p := range 8 {
		shift := 8 * p
		if hist[p][byte(key(src[0])>>shift)] == n {
			continue
		}
		var pos [256]int
		for b, sum := 0, 0; b < 256; b++ {
			pos[b], sum = sum, sum+hist[p][b]
		}
		for _, x := range src { // in input order: stable
			d := byte(key(x) >> shift)
			dst[pos[d]] = x
			pos[d]++
		}
		src, dst, run = dst, src, run+1
	}
	if run%2 == 1 {
		copy(xs, src)
	}
	return info{path: radixPath, passes: run}
}

// ============================ the prover ============================

var rng = rand.New(rand.NewPCG(1, 2))

func u(m uint64) uint64 { return rng.Uint64() % m }

// genU64: half the cases on a narrow range (k <= 8, ties everywhere), half
// with random bytes masked to zero (so some passes can be skipped).
func genU64(i, n int) []uint64 {
	xs := make([]uint64, n)
	if i%2 == 1 {
		base, k := rng.Uint64(), 1+u(8)
		for j := range xs {
			xs[j] = base + u(k)
		}
		return xs
	}
	m, mask := rng.Uint64(), uint64(0)
	for p := range 8 {
		if m>>p&1 == 1 {
			mask |= 0xFF << (8 * p)
		}
	}
	for j := range xs {
		xs[j] = rng.Uint64() & mask
	}
	return xs
}

var specials = []uint64{0, 1 << 63, 0x7FF0000000000000, 0xFFF0000000000000, // +-0, +-Inf
	0x7FF8000000000000, 0xFFF8000000000000, 0x7FF0000000000001, // +-NaN, a NaN payload
	1, 1<<63 | 1, 0x0010000000000000, 0x7FEFFFFFFFFFFFFF, // +-5e-324, min normal, max
	0x3FF0000000000000, 0xBFF0000000000000} // +-1

func randFloat() float64 {
	switch u(3) {
	case 0:
		return math.Float64frombits(specials[u(uint64(len(specials)))])
	case 1:
		return math.Float64frombits(rng.Uint64()) // any bit pattern
	}
	return float64(int64(u(2001))-1000) / 4 // exact small values, many ties
}

func genF64(i, n int) []float64 {
	pool := []float64{} // narrow: up to 8 distinct values, repeated
	for range (1 + u(8)) * uint64(i%2) {
		pool = append(pool, randFloat())
	}
	xs := make([]float64, n)
	for j := range xs {
		if xs[j] = randFloat(); i%2 == 1 {
			xs[j] = pool[u(uint64(len(pool)))]
		}
	}
	return xs
}

var tokens = []string{"a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k", "l", "m", "n", "o",
	"p", "q", "r", "s", "t", "u", "v", "w", "x", "y", "z", "A", "Z", "0", "\x00", "é", "€"}

func word(alpha, maxLen uint64) (s string) {
	for range u(maxLen + 1) {
		s += tokens[u(alpha)]
	}
	return s
}

func genStr(i, n int) []string {
	xs, pre := make([]string, n), word(32, 7)
	for j := range xs {
		switch {
		case i%2 == 1:
			xs[j] = word(2, 4) // "a"/"b", length 0..4: duplicates and prefixes
		case u(2) == 0:
			xs[j] = pre + word(32, 12) // a shared prefix
		default:
			xs[j] = word(32, 12)
		}
	}
	return xs
}

func genI64(i, n int) []int64 { return conv(genU64(i, n), func(x uint64) int64 { return int64(x) }) }

type rec struct {
	key uint64
	tag int // input position: equal keys must keep it increasing
}

func tagged(keys []uint64) []rec {
	xs := make([]rec, len(keys))
	for j, k := range keys {
		xs[j] = rec{k, j}
	}
	return xs
}

func genRec(i, n int) []rec { return tagged(genU64(i, n)) }

func descending(n int) []uint64 {
	xs := make([]uint64, n)
	for i := range xs {
		xs[i] = uint64(n - 1 - i)
	}
	return xs
}

func conv[A, B any](xs []A, f func(A) B) []B {
	ys := make([]B, len(xs))
	for i, x := range xs {
		ys[i] = f(x)
	}
	return ys
}

var (
	desc     = descending(1000)
	bits     = func(bs ...uint64) []float64 { return conv(bs, math.Float64frombits) }
	u64Edges = [][]uint64{{}, {7}, slices.Repeat([]uint64{42}, 1000), desc, {0, math.MaxUint64, 1, math.MaxUint64 - 1, 0}}
	i64Edges = [][]int64{{}, {-1}, slices.Repeat([]int64{-7}, 1000), conv(desc, func(x uint64) int64 { return int64(x) - 500 }),
		{math.MinInt64, math.MaxInt64, 0, -1, 1, math.MinInt64, math.MaxInt64}}
	f64Edges = [][]float64{{}, bits(specials[4]), bits(slices.Concat(specials, specials, specials)...),
		bits(1, 2, 3, 1<<63|1, 1<<63|2, 0, 1<<63, 0x000FFFFFFFFFFFFF, 1<<63|0x000FFFFFFFFFFFFF, 1),
		slices.Repeat(bits(1<<63, 0), 300)} // -0 and +0, interleaved
	strEdges = [][]string{{}, {""}, {"", "", ""}, {"she", "shells", "s", "", "sh", "shell", "she", "shells"},
		slices.Repeat([]string{"same"}, 100), conv(desc[:300], func(x uint64) string { return fmt.Sprintf("https://example.com/u/%06d", x%37) })}
	recEdges = [][]rec{{}, {{5, 0}}, tagged(slices.Repeat([]uint64{42}, 1000)), tagged(desc), tagged(u64Edges[4])}
)

type tally struct {
	cases, fail int
	paths       [3]int // cases that took the stdlib, counting and radix path
}

// prove runs one function over random cases, then over the edge cases, at the
// given threshold. Equal to the oracle's sort of the same input implies that
// the output is also a permutation of that input.
func prove[T any](cases, maxLen, small int, edges [][]T, gen func(i, n int) []T,
	sortFn func([]T, int) info, oracle func([]T), eq func(a, b T) bool) (t tally) {
	check := func(in []T) {
		want := slices.Clone(in)
		t.paths[sortFn(in, small).path]++
		oracle(want)
		t.cases++
		if !slices.EqualFunc(in, want, eq) {
			t.fail++
		}
	}
	for i := range cases {
		check(gen(i, int(u(uint64(maxLen+1)))))
	}
	for _, e := range edges {
		check(slices.Clone(e))
	}
	return t
}

func same[T comparable](a, b T) bool { return a == b }

// totalOrderOracle is written independently of floatKey: by cases on the sign.
func totalOrderOracle(a, b float64) int {
	x, y := math.Float64bits(a), math.Float64bits(b)
	switch {
	case x>>63 != y>>63: // different signs: the negative one first
		return cmp.Compare(y>>63, x>>63)
	case x>>63 == 0: // both non-negative: bigger bits, bigger value
		return cmp.Compare(x, y)
	default: // both negative: bigger bits, smaller value
		return cmp.Compare(y, x)
	}
}

func sortU64(xs []uint64, small int) info { return sortInts(xs, 0, small) }
func sortI64(xs []int64, small int) info  { return sortInts(xs, 1<<63, small) }
func byKey(xs []rec, small int) info {
	return sortByKey(xs, func(r rec) uint64 { return r.key }, small)
}

func proveAll(cases, maxLen, small int) {
	show := func(name string, t tally) {
		fmt.Printf("  %-12s %7d %7d %9d %7d %9d\n", name, t.cases, t.paths[0], t.paths[1], t.paths[2], t.fail)
	}
	fmt.Printf("  %-12s %7s %7s %9s %7s %9s\n", "function", "cases", "stdlib", "counting", "radix", "failures")
	show("SortUint64", prove(cases, maxLen, small, u64Edges, genU64, sortU64, slices.Sort, same))
	show("SortInt64", prove(cases, maxLen, small, i64Edges, genI64, sortI64, slices.Sort, same))
	show("SortFloat64", prove(cases, maxLen, small, f64Edges, genF64, sortFloats,
		func(xs []float64) { slices.SortFunc(xs, totalOrderOracle) },
		func(a, b float64) bool { return math.Float64bits(a) == math.Float64bits(b) })) // NaN != NaN
	show("SortStrings", prove(cases, maxLen, small, strEdges, genStr, sortStrings, slices.Sort, same))
	show("SortByKey", prove(cases, maxLen, small, recEdges, genRec, byKey,
		func(xs []rec) { slices.SortStableFunc(xs, func(a, b rec) int { return cmp.Compare(a.key, b.key) }) }, same))
}

func main() {
	fmt.Println("1. Prove: threshold forced to 0, so every case of n >= 2 runs our code.")
	fmt.Println("   20,000 random cases per function, n = 0..300, half on a narrow range, + edge cases.")
	fmt.Println("   Oracles: slices.Sort; a by-cases total-order comparator (floats, compared as bits);")
	fmt.Println("   slices.SortStableFunc (SortByKey: equal keys must keep their input order).")
	proveAll(20000, 300, 0)
	fmt.Printf("\n2. The public API (stdlib below n = %d): 200 random cases, n = 0..%d, + edge cases.\n", smallN, 4*smallN)
	proveAll(200, 4*smallN, smallN)
	measure()
}

// ============================ the benchmark ============================

func nsPerElem(n int, f func()) float64 {
	runs, start := 0, time.Now()
	for runs < 3 || time.Since(start) < 300*time.Millisecond {
		f()
		runs++
	}
	return float64(time.Since(start).Nanoseconds()) / float64(runs) / float64(n)
}

func describe(in info) string {
	switch in.path {
	case countPath:
		return fmt.Sprintf("counting, k = %d", in.k)
	case radixPath:
		if in.passes == 0 {
			return "MSD, cutoff 32"
		}
		return fmt.Sprintf("LSD, %d of 8 passes", in.passes)
	}
	return "stdlib"
}

// bench checks the library against the stdlib on in, then times both; each
// run copies the pristine input first, and the copy alone is timed too.
func bench[T cmp.Ordered](name string, in []T, lib func([]T, int) info) {
	got, want := slices.Clone(in), slices.Clone(in)
	what := describe(lib(got, smallN))
	if slices.Sort(want); !slices.Equal(got, want) {
		panic(name + ": library != stdlib")
	}
	w := make([]T, len(in))
	allocs := testing.AllocsPerRun(1, func() { copy(w, in); lib(w, smallN) })
	cp := nsPerElem(len(in), func() { copy(w, in) })
	l := nsPerElem(len(in), func() { copy(w, in); lib(w, smallN) })
	s := nsPerElem(len(in), func() { copy(w, in); slices.Sort(w) })
	fmt.Printf("  %-19s %-20s %6.0f %8.1f %8.1f %5.1f %7.1fx\n", name, what, allocs, l, s, cp, (s-cp)/(l-cp))
}

func measure() {
	const n = 1_000_000
	r := rand.New(rand.NewPCG(3, 4))
	random, stamps, small := make([]uint64, n), make([]uint64, n), make([]int64, n)
	floats, urls, ends := make([]float64, n), make([]string, n), make([]int, n)
	var all strings.Builder // every URL's bytes in one block; each URL is a slice of it
	all.Grow(52 * n)        // a URL here is at most 52 bytes: one allocation, no regrowth
	day := uint64(time.Date(2026, 11, 9, 0, 0, 0, 0, time.UTC).UnixNano())
	for i := range n {
		random[i] = r.Uint64()
		stamps[i] = day + r.Uint64()%(86_400*1_000_000_000)
		small[i] = int64(r.Uint64()%1000) - 500
		floats[i] = float64(int64(r.Uint64()>>11)-1<<52) / 1024 // finite, no -0, no NaN
		fmt.Fprintf(&all, "https://example.com/api/v1/users/%06d/orders/%d", r.Uint64()%1_000_000, r.Uint64()%100_000)
		ends[i] = all.Len()
	}
	for i, block, start := 0, all.String(), 0; i < n; i++ {
		urls[i], start = block[start:ends[i]], ends[i]
	}
	fmt.Printf("\n3. Measure at n = %d (each output checked equal to slices.Sort's first).\n", n)
	fmt.Println("   ns/element, each run including a copy of the pristine input (the copy column);")
	fmt.Println("   allocs = heap allocations per library call; speedup = (stdlib-copy)/(library-copy)")
	fmt.Printf("  %-19s %-20s %6s %8s %8s %5s %8s\n", "workload", "library path", "allocs", "library", "stdlib", "copy", "speedup")
	bench("random uint64", random, sortU64)
	bench("timestamps (1 day)", stamps, sortU64)
	bench("ints in [-500,500)", small, sortI64)
	bench("random float64", floats, sortFloats)
	bench("URLs", urls, sortStrings)
}
```

**Sample output:**

```
1. Prove: threshold forced to 0, so every case of n >= 2 runs our code.
   20,000 random cases per function, n = 0..300, half on a narrow range, + edge cases.
   Oracles: slices.Sort; a by-cases total-order comparator (floats, compared as bits);
   slices.SortStableFunc (SortByKey: equal keys must keep their input order).
  function       cases  stdlib  counting   radix  failures
  SortUint64     20005     138      9944    9923         0
  SortInt64      20005     132      9941    9932         0
  SortFloat64    20005     140      1246   18619         0
  SortStrings    20006     132         0   19874         0
  SortByKey      20005     134         0   19871         0

2. The public API (stdlib below n = 1024): 200 random cases, n = 0..4096, + edge cases.
  function       cases  stdlib  counting   radix  failures
  SortUint64       205      51        79      75         0
  SortInt64        205      48        80      77         0
  SortFloat64      205      47        12     146         0
  SortStrings      206      46         0     160         0
  SortByKey        205      65         0     140         0

3. Measure at n = 1000000 (each output checked equal to slices.Sort's first).
   ns/element, each run including a copy of the pristine input (the copy column);
   allocs = heap allocations per library call; speedup = (stdlib-copy)/(library-copy)
  workload            library path         allocs  library   stdlib  copy  speedup
  random uint64       LSD, 8 of 8 passes        1      8.7     58.1   0.1     6.8x
  timestamps (1 day)  LSD, 6 of 8 passes        1      7.6     65.3   0.2     8.8x
  ints in [-500,500)  counting, k = 1000        1      4.3     35.8   0.2     8.7x
  random float64      LSD, 8 of 8 passes        2     10.8     73.6   0.1     6.9x
  URLs                MSD, cutoff 32            1    100.8    173.2   0.4     1.7x
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::fmt::Write;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// ===================== mod radix (one file here) =====================
//
// sort (generic over u64 / i64 / f64 through a small trait), sort_strings and
// the stable sort_by_radix_key, assembled from the lesson: order-preserving
// key transforms (ex 8), one read for all eight histograms and pass skipping
// (ex 13), counting sort when the range is at most n (ex 4), MSD with a cutoff
// for strings (ex 9), one buffer per call that every pass reuses, and the
// stdlib below a threshold (ex 12).

// SMALL_N: below this, defer to the stdlib. Example 12's Rust twin measured
// lsd8 first beating sort_unstable between n = 16K and 64K (ipnsort is much
// faster than Go's pdqsort there); we take the middle as the constant.
const SMALL_N: usize = 32_768;

const STR_CUTOFF: usize = 32; // MSD buckets this small are finished by insertion sort

#[derive(Clone, Copy)]
enum Path {
    Std,
    Count,
    Radix,
}

/// What a call did (the prover and the benchmark print it).
struct Info {
    path: Path,
    passes: usize, // LSD byte passes actually run, of 8
    k: u64,        // counting sort: the range max-min+1
}

impl Info {
    fn of(path: Path) -> Self {
        Info {
            path,
            passes: 0,
            k: 0,
        }
    }
}

/// A key that maps to u64 without changing its order, and back. Go needs a
/// function per type (and a separate key slice for floats); here one generic
/// core is monomorphised per type, so the transform is inlined into each pass.
pub trait RadixKey: Copy {
    fn key(self) -> u64;
    fn from_key(k: u64) -> Self;
}

impl RadixKey for u64 {
    fn key(self) -> u64 {
        self
    }
    fn from_key(k: u64) -> Self {
        k
    }
}

impl RadixKey for i64 {
    fn key(self) -> u64 {
        (self as u64) ^ 1 << 63 // flip the sign bit (ex 8)
    }
    fn from_key(k: u64) -> Self {
        (k ^ 1 << 63) as i64
    }
}

impl RadixKey for f64 {
    // IEEE totalOrder: negatives flip every bit, non-negatives the sign bit.
    // This is how std's f64::total_cmp is written too (read in the source).
    fn key(self) -> u64 {
        let b = self.to_bits();
        b ^ (((b as i64 >> 63) as u64) | 1 << 63)
    }
    fn from_key(k: u64) -> Self {
        f64::from_bits(k ^ (((!k as i64 >> 63) as u64) | 1 << 63))
    }
}

pub fn sort<T: RadixKey>(xs: &mut [T]) {
    sort_with(xs, SMALL_N);
}

pub fn sort_strings<S: AsRef<[u8]> + Default>(xs: &mut [S]) {
    sort_strings_with(xs, SMALL_N);
}

/// Sorts stably by key(x), ascending.
pub fn sort_by_radix_key<T: Default>(xs: &mut [T], key: impl Fn(&T) -> u64) {
    sort_by_key_with(xs, key, SMALL_N);
}

fn sort_with<T: RadixKey>(xs: &mut [T], small: usize) -> Info {
    let n = xs.len();
    if n < small.max(2) {
        // f64 is not Ord, so the generic fallback sorts by the key: the same
        // order as the radix path for all three types (total_cmp for floats).
        xs.sort_unstable_by_key(|x| x.key());
        return Info::of(Path::Std);
    }
    let mut hist = [[0usize; 256]; 8]; // one read: min, max and every histogram
    let (mut lo, mut hi) = (u64::MAX, 0u64);
    for &x in xs.iter() {
        let k = x.key();
        (lo, hi) = (lo.min(k), hi.max(k));
        for (p, h) in hist.iter_mut().enumerate() {
            h[(k >> (8 * p)) as u8 as usize] += 1; // a u8 index: no bounds check
        }
    }
    if hi - lo < n as u64 {
        // range <= n: counting sort, rewrite from counts
        let mut counts = vec![0usize; (hi - lo + 1) as usize];
        for &x in xs.iter() {
            counts[(x.key() - lo) as usize] += 1;
        }
        let mut i = 0;
        for (v, &c) in (0u64..).zip(&counts) {
            xs[i..i + c].fill(T::from_key(lo + v));
            i += c;
        }
        return Info {
            path: Path::Count,
            passes: 0,
            k: hi - lo + 1,
        };
    }
    let mut buf = xs.to_vec(); // the one buffer, reused by every pass
    let (mut src, mut dst): (&mut [T], &mut [T]) = (&mut *xs, &mut buf);
    let mut run = 0;
    for (p, h) in hist.iter().enumerate() {
        let shift = 8 * p;
        if h[(src[0].key() >> shift) as u8 as usize] == n {
            continue; // every key has the same digit here: skip the pass
        }
        let mut pos = [0usize; 256];
        let mut sum = 0;
        for (q, &c) in pos.iter_mut().zip(h) {
            (*q, sum) = (sum, sum + c);
        }
        for &x in src.iter() {
            let d = (x.key() >> shift) as u8 as usize;
            dst[pos[d]] = x;
            pos[d] += 1;
        }
        std::mem::swap(&mut src, &mut dst);
        run += 1;
    }
    if run % 2 == 1 {
        xs.copy_from_slice(&buf); // an odd number of passes ends in buf
    }
    Info {
        path: Path::Radix,
        passes: run,
        k: 0,
    }
}

fn sort_strings_with<S: AsRef<[u8]> + Default>(xs: &mut [S], small: usize) -> Info {
    if xs.len() < small.max(2) {
        xs.sort_unstable_by(|a, b| a.as_ref().cmp(b.as_ref()));
        return Info::of(Path::Std);
    }
    // Default plays the part of Go's zero value; the strings themselves are
    // MOVED into the buffer and back (mem::take), never cloned.
    let mut buf: Vec<S> = (0..xs.len()).map(|_| S::default()).collect();
    msd(xs, &mut buf, 0);
    Info::of(Path::Radix)
}

/// 0 = end of string (below every byte), else byte+1. Option replaces Go's test.
fn char_at(s: &[u8], d: usize) -> usize {
    s.get(d).map_or(0, |&b| b as usize + 1)
}

/// Sorts xs, whose strings all share their first d bytes.
fn msd<S: AsRef<[u8]> + Default>(xs: &mut [S], buf: &mut [S], mut d: usize) {
    loop {
        if xs.len() <= STR_CUTOFF {
            for i in 1..xs.len() {
                // insertion sort on the suffixes
                let mut j = i;
                while j > 0 && xs[j].as_ref()[d..] < xs[j - 1].as_ref()[d..] {
                    xs.swap(j, j - 1);
                    j -= 1;
                }
            }
            return;
        }
        let mut counts = [0usize; 257];
        for s in xs.iter() {
            counts[char_at(s.as_ref(), d)] += 1;
        }
        let c = char_at(xs[0].as_ref(), d);
        if counts[c] == xs.len() {
            // one bucket holds all
            if c == 0 {
                return; // all strings ended: all equal
            }
            d += 1; // a shared byte: skip it without moving anything
            continue;
        }
        let mut pos = [0usize; 257];
        let mut sum = 0;
        for (q, &k) in pos.iter_mut().zip(&counts) {
            (*q, sum) = (sum, sum + k);
        }
        for s in xs.iter_mut() {
            let c = char_at(s.as_ref(), d);
            buf[pos[c]] = std::mem::take(s);
            pos[c] += 1;
        }
        xs.swap_with_slice(&mut buf[..xs.len()]); // move them all back
        let mut start = counts[0]; // bucket 0 is done
        for &k in &counts[1..] {
            if k > 1 {
                let r = start..start + k;
                msd(&mut xs[r.clone()], &mut buf[r], d + 1);
            }
            start += k;
        }
        return;
    }
}

fn sort_by_key_with<T: Default>(xs: &mut [T], key: impl Fn(&T) -> u64, small: usize) -> Info {
    let n = xs.len();
    if n < small.max(2) {
        xs.sort_by_key(&key); // driftsort: stable (and it allocates)
        return Info::of(Path::Std);
    }
    let mut hist = [[0usize; 256]; 8];
    for x in xs.iter() {
        let k = key(x);
        for (p, h) in hist.iter_mut().enumerate() {
            h[(k >> (8 * p)) as u8 as usize] += 1;
        }
    }
    let mut buf: Vec<T> = (0..n).map(|_| T::default()).collect();
    let (mut src, mut dst): (&mut [T], &mut [T]) = (&mut *xs, &mut buf);
    let mut run = 0;
    for (p, h) in hist.iter().enumerate() {
        let shift = 8 * p;
        if h[(key(&src[0]) >> shift) as u8 as usize] == n {
            continue;
        }
        let mut pos = [0usize; 256];
        let mut sum = 0;
        for (q, &c) in pos.iter_mut().zip(h) {
            (*q, sum) = (sum, sum + c);
        }
        for x in src.iter_mut() {
            // in input order: stable
            let d = (key(x) >> shift) as u8 as usize;
            dst[pos[d]] = std::mem::take(x);
            pos[d] += 1;
        }
        std::mem::swap(&mut src, &mut dst);
        run += 1;
    }
    if run % 2 == 1 {
        xs.swap_with_slice(&mut buf);
    }
    Info {
        path: Path::Radix,
        passes: run,
        k: 0,
    }
}

// ============================ the prover ============================

// Go's math/rand/v2 PCG (PCG-DXSM), so every case is the one Go generates.
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

    fn u(&mut self, m: u64) -> u64 {
        self.next_u64() % m
    }
}

// half the cases on a narrow range (k <= 8, ties everywhere), half with
// random bytes masked to zero (so some passes can be skipped).
fn gen_u64(r: &mut Pcg, i: usize, n: usize) -> Vec<u64> {
    if i % 2 == 1 {
        let (base, k) = (r.next_u64(), 1 + r.u(8));
        return (0..n).map(|_| base.wrapping_add(r.u(k))).collect();
    }
    let m = r.next_u64();
    let mask = (0..8)
        .filter(|p| m >> p & 1 == 1)
        .fold(0u64, |acc, p| acc | 0xFF << (8 * p));
    (0..n).map(|_| r.next_u64() & mask).collect()
}

const SPECIALS: [u64; 13] = [
    0,
    1 << 63,
    0x7FF0000000000000,
    0xFFF0000000000000, // +-0, +-Inf
    0x7FF8000000000000,
    0xFFF8000000000000,
    0x7FF0000000000001, // +-NaN, a NaN payload
    1,
    1 << 63 | 1,
    0x0010000000000000,
    0x7FEFFFFFFFFFFFFF, // +-5e-324, min normal, max
    0x3FF0000000000000,
    0xBFF0000000000000, // +-1
];

fn rand_float(r: &mut Pcg) -> f64 {
    match r.u(3) {
        0 => f64::from_bits(SPECIALS[r.u(SPECIALS.len() as u64) as usize]),
        1 => f64::from_bits(r.next_u64()), // any bit pattern
        _ => (r.u(2001) as i64 - 1000) as f64 / 4.0, // exact small values, many ties
    }
}

fn gen_f64(r: &mut Pcg, i: usize, n: usize) -> Vec<f64> {
    let count = (1 + r.u(8)) * (i % 2) as u64; // narrow: up to 8 distinct values
    let pool: Vec<f64> = (0..count).map(|_| rand_float(r)).collect();
    (0..n)
        .map(|_| {
            let x = rand_float(r);
            if i % 2 == 1 {
                pool[r.u(pool.len() as u64) as usize]
            } else {
                x
            }
        })
        .collect()
}

const TOKENS: [&str; 32] = [
    "a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k", "l", "m", "n", "o", "p", "q", "r", "s",
    "t", "u", "v", "w", "x", "y", "z", "A", "Z", "0", "\x00", "é", "€",
];

fn word(r: &mut Pcg, alpha: u64, max_len: u64) -> String {
    (0..r.u(max_len + 1))
        .map(|_| TOKENS[r.u(alpha) as usize])
        .collect()
}

fn gen_str(r: &mut Pcg, i: usize, n: usize) -> Vec<String> {
    let pre = word(r, 32, 7);
    (0..n)
        .map(|_| {
            if i % 2 == 1 {
                word(r, 2, 4) // "a"/"b", length 0..4: duplicates and prefixes
            } else if r.u(2) == 0 {
                pre.clone() + &word(r, 32, 12) // a shared prefix
            } else {
                word(r, 32, 12)
            }
        })
        .collect()
}

fn gen_i64(r: &mut Pcg, i: usize, n: usize) -> Vec<i64> {
    gen_u64(r, i, n).into_iter().map(|x| x as i64).collect()
}

#[derive(Clone, Copy, Default, PartialEq)]
struct Rec {
    key: u64,
    tag: usize, // input position: equal keys must keep it increasing
}

fn tagged(keys: &[u64]) -> Vec<Rec> {
    (0..)
        .zip(keys)
        .map(|(tag, &key)| Rec { key, tag })
        .collect()
}

fn gen_rec(r: &mut Pcg, i: usize, n: usize) -> Vec<Rec> {
    tagged(&gen_u64(r, i, n))
}

#[derive(Default)]
struct Tally {
    cases: usize,
    fail: usize,
    paths: [usize; 3], // cases that took the stdlib, counting and radix path
}

// One function over random cases, then the edge cases, at the given threshold.
// Equal to the oracle's sort of the same input implies the output is also a
// permutation of that input.
#[allow(clippy::too_many_arguments)] // the eight knobs of one property test
fn prove<T: Clone>(
    r: &mut Pcg,
    cases: usize,
    max_len: usize,
    small: usize,
    edges: &[Vec<T>],
    generate: fn(&mut Pcg, usize, usize) -> Vec<T>,
    sort_fn: impl Fn(&mut [T], usize) -> Info,
    oracle: impl Fn(&mut [T]),
    eq: impl Fn(&T, &T) -> bool,
) -> Tally {
    let mut t = Tally::default();
    let mut check = |mut got: Vec<T>| {
        let mut want = got.clone();
        t.paths[sort_fn(&mut got, small).path as usize] += 1;
        oracle(&mut want);
        t.cases += 1;
        if !got.iter().zip(&want).all(|(a, b)| eq(a, b)) {
            t.fail += 1;
        }
    };
    for i in 0..cases {
        let n = r.u(max_len as u64 + 1) as usize;
        check(generate(r, i, n));
    }
    for e in edges {
        check(e.clone());
    }
    t
}

struct Edges {
    u64s: Vec<Vec<u64>>,
    i64s: Vec<Vec<i64>>,
    f64s: Vec<Vec<f64>>,
    strs: Vec<Vec<String>>,
    recs: Vec<Vec<Rec>>,
}

fn edges() -> Edges {
    let desc: Vec<u64> = (0..1000).rev().collect();
    let bits = |bs: &[u64]| bs.iter().map(|&b| f64::from_bits(b)).collect::<Vec<_>>();
    let u64s = vec![
        vec![],
        vec![7],
        vec![42; 1000],
        desc.clone(),
        vec![0, u64::MAX, 1, u64::MAX - 1, 0],
    ];
    let strs: Vec<Vec<&str>> = vec![
        vec![],
        vec![""],
        vec!["", "", ""],
        vec!["she", "shells", "s", "", "sh", "shell", "she", "shells"],
        vec!["same"; 100],
    ];
    let mut strs: Vec<Vec<String>> = strs
        .into_iter()
        .map(|v| v.into_iter().map(String::from).collect())
        .collect();
    strs.push(
        desc[..300]
            .iter()
            .map(|x| format!("https://example.com/u/{:06}", x % 37))
            .collect(),
    );
    const M: u64 = 0x000FFFFFFFFFFFFF; // largest subnormal
    Edges {
        i64s: vec![
            vec![],
            vec![-1],
            vec![-7; 1000],
            desc.iter().map(|&x| x as i64 - 500).collect(),
            vec![i64::MIN, i64::MAX, 0, -1, 1, i64::MIN, i64::MAX],
        ],
        f64s: vec![
            vec![],
            bits(&[SPECIALS[4]]),
            bits(&SPECIALS.repeat(3)),
            bits(&[
                1,
                2,
                3,
                1 << 63 | 1,
                1 << 63 | 2,
                0,
                1 << 63,
                M,
                1 << 63 | M,
                1,
            ]),
            bits(&[1 << 63, 0].repeat(300)), // -0 and +0, interleaved
        ],
        recs: vec![
            vec![],
            vec![Rec { key: 5, tag: 0 }],
            tagged(&[42; 1000]),
            tagged(&desc),
            tagged(&u64s[4]),
        ],
        u64s,
        strs,
    }
}

fn prove_all(r: &mut Pcg, e: &Edges, cases: usize, max_len: usize, small: usize) {
    let show = |name: &str, t: Tally| {
        println!(
            "  {name:<17} {:>7} {:>7} {:>9} {:>7} {:>9}",
            t.cases, t.paths[0], t.paths[1], t.paths[2], t.fail
        );
    };
    println!(
        "  {:<17} {:>7} {:>7} {:>9} {:>7} {:>9}",
        "function", "cases", "stdlib", "counting", "radix", "failures"
    );
    let (c, m, s) = (cases, max_len, small);
    let t = prove(
        r,
        c,
        m,
        s,
        &e.u64s,
        gen_u64,
        sort_with,
        <[u64]>::sort_unstable,
        u64::eq,
    );
    show("sort::<u64>", t);
    let t = prove(
        r,
        c,
        m,
        s,
        &e.i64s,
        gen_i64,
        sort_with,
        <[i64]>::sort_unstable,
        i64::eq,
    );
    show("sort::<i64>", t);
    let f64_oracle = |xs: &mut [f64]| xs.sort_unstable_by(f64::total_cmp);
    let bits_eq = |a: &f64, b: &f64| a.to_bits() == b.to_bits(); // NaN != NaN
    let t = prove(r, c, m, s, &e.f64s, gen_f64, sort_with, f64_oracle, bits_eq);
    show("sort::<f64>", t);
    let t = prove(
        r,
        c,
        m,
        s,
        &e.strs,
        gen_str,
        sort_strings_with,
        <[String]>::sort_unstable,
        String::eq,
    );
    show("sort_strings", t);
    let by_key = |xs: &mut [Rec], small| sort_by_key_with(xs, |r: &Rec| r.key, small);
    let rec_oracle = |xs: &mut [Rec]| xs.sort_by_key(|r| r.key);
    let t = prove(r, c, m, s, &e.recs, gen_rec, by_key, rec_oracle, Rec::eq);
    show("sort_by_radix_key", t);
}

// ---- allocation counter (stand-in for testing.AllocsPerRun) ----
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

fn main() {
    let mut r = Pcg::new(1, 2);
    let e = edges();
    println!("1. Prove: threshold forced to 0, so every case of n >= 2 runs our code.");
    println!(
        "   20,000 random cases per function, n = 0..300, half on a narrow range, + edge cases."
    );
    println!("   Oracles: sort_unstable; f64::total_cmp (floats, compared as bits);");
    println!("   [T]::sort_by_key (sort_by_radix_key: equal keys must keep their input order).");
    prove_all(&mut r, &e, 20000, 300, 0);
    println!(
        "\n2. The public API (stdlib below n = {SMALL_N}): 200 random cases, n = 0..{}, + edge cases.",
        4 * SMALL_N
    );
    prove_all(&mut r, &e, 200, 4 * SMALL_N, SMALL_N);
    measure();
}

// ============================ the benchmark ============================

fn ns_per_elem(n: usize, mut f: impl FnMut()) -> f64 {
    let (mut runs, start) = (0u32, Instant::now());
    while runs < 3 || start.elapsed() < Duration::from_millis(300) {
        f();
        runs += 1;
    }
    start.elapsed().as_nanos() as f64 / f64::from(runs) / n as f64
}

fn describe(info: &Info) -> String {
    match info.path {
        Path::Count => format!("counting, k = {}", info.k),
        Path::Radix if info.passes == 0 => "MSD, cutoff 32".to_string(),
        Path::Radix => format!("LSD, {} of 8 passes", info.passes),
        Path::Std => "stdlib".to_string(),
    }
}

/// Checks the library against the stdlib on `input`, then times both; each run
/// copies the pristine input first, and the copy alone is timed too.
fn bench<T: Copy + PartialEq>(
    name: &str,
    input: &[T],
    lib: impl Fn(&mut [T], usize) -> Info,
    stdlib: impl Fn(&mut [T]),
) {
    let (mut got, mut want) = (input.to_vec(), input.to_vec());
    let what = describe(&lib(&mut got, SMALL_N));
    stdlib(&mut want);
    assert!(got == want, "{name}: library != stdlib");
    drop((got, want)); // keep the peak memory down while timing
    let mut w = input.to_vec();
    let a0 = ALLOCS.load(Relaxed);
    w.copy_from_slice(input);
    lib(&mut w, SMALL_N);
    let allocs = ALLOCS.load(Relaxed) - a0;
    let n = input.len();
    let cp = ns_per_elem(n, || w.copy_from_slice(black_box(input)));
    let l = ns_per_elem(n, || {
        w.copy_from_slice(input);
        lib(black_box(&mut w), SMALL_N);
    });
    let s = ns_per_elem(n, || {
        w.copy_from_slice(input);
        stdlib(black_box(&mut w));
    });
    let speedup = (s - cp) / (l - cp);
    println!("  {name:<19} {what:<20} {allocs:>6} {l:>8.1} {s:>8.1} {cp:>5.1} {speedup:>7.1}x");
}

fn measure() {
    const N: usize = 1_000_000;
    const DAY: u64 = 1_794_182_400_000_000_000; // 2026-11-09T00:00:00Z in ns (std has no calendar)
    let mut r = Pcg::new(3, 4);
    let (mut random, mut stamps, mut small) = (vec![0u64; N], vec![0u64; N], vec![0i64; N]);
    let mut floats = vec![0f64; N];
    // every URL's bytes in one block; each URL is a &str into it (as in Go)
    // (a URL here is at most 52 bytes: one allocation, no regrowth)
    let (mut all, mut ends) = (String::with_capacity(52 * N), Vec::with_capacity(N));
    for i in 0..N {
        random[i] = r.next_u64();
        stamps[i] = DAY + r.u(86_400 * 1_000_000_000);
        small[i] = r.u(1000) as i64 - 500;
        floats[i] = ((r.next_u64() >> 11) as i64 - (1 << 52)) as f64 / 1024.0; // finite, no -0, no NaN
        let (user, order) = (r.u(1_000_000), r.u(100_000));
        write!(
            all,
            "https://example.com/api/v1/users/{user:06}/orders/{order}"
        )
        .unwrap();
        ends.push(all.len());
    }
    let starts = std::iter::once(0).chain(ends.iter().copied());
    let url_refs: Vec<&str> = starts.zip(&ends).map(|(a, &b)| &all[a..b]).collect();
    println!("\n3. Measure at n = {N} (each output checked equal to sort_unstable's first).");
    println!("   ns/element, each run including a copy of the pristine input (the copy column);");
    println!(
        "   allocs = heap allocations per library call; speedup = (stdlib-copy)/(library-copy)"
    );
    println!(
        "  {:<19} {:<20} {:>6} {:>8} {:>8} {:>5} {:>8}",
        "workload", "library path", "allocs", "library", "stdlib", "copy", "speedup"
    );
    bench("random uint64", &random, sort_with, <[u64]>::sort_unstable);
    bench(
        "timestamps (1 day)",
        &stamps,
        sort_with,
        <[u64]>::sort_unstable,
    );
    bench(
        "ints in [-500,500)",
        &small,
        sort_with,
        <[i64]>::sort_unstable,
    );
    bench("random float64", &floats, sort_with, |x: &mut [f64]| {
        x.sort_unstable_by(f64::total_cmp)
    });
    bench(
        "URLs",
        &url_refs,
        sort_strings_with,
        <[&str]>::sort_unstable,
    );
}
```

**Rust sample output:**

```
1. Prove: threshold forced to 0, so every case of n >= 2 runs our code.
   20,000 random cases per function, n = 0..300, half on a narrow range, + edge cases.
   Oracles: sort_unstable; f64::total_cmp (floats, compared as bits);
   [T]::sort_by_key (sort_by_radix_key: equal keys must keep their input order).
  function            cases  stdlib  counting   radix  failures
  sort::<u64>         20005     138      9944    9923         0
  sort::<i64>         20005     132      9941    9932         0
  sort::<f64>         20005     140      1246   18619         0
  sort_strings        20006     132         0   19874         0
  sort_by_radix_key   20005     134         0   19871         0

2. The public API (stdlib below n = 32768): 200 random cases, n = 0..131072, + edge cases.
  function            cases  stdlib  counting   radix  failures
  sort::<u64>           205      64        68      73         0
  sort::<i64>           205      59        74      72         0
  sort::<f64>           205      62        12     131         0
  sort_strings          206      63         0     143         0
  sort_by_radix_key     205      56         0     149         0

3. Measure at n = 1000000 (each output checked equal to sort_unstable's first).
   ns/element, each run including a copy of the pristine input (the copy column);
   allocs = heap allocations per library call; speedup = (stdlib-copy)/(library-copy)
  workload            library path         allocs  library   stdlib  copy  speedup
  random uint64       LSD, 8 of 8 passes        1      5.9     10.1   0.2     1.7x
  timestamps (1 day)  LSD, 6 of 8 passes        1      5.1     11.1   0.2     2.2x
  ints in [-500,500)  counting, k = 1000        1      3.3      4.9   0.2     1.5x
  random float64      LSD, 8 of 8 passes        1      9.4     13.0   0.2     1.4x
  URLs                MSD, cutoff 32            1    108.2     92.6   0.5     0.9x
```

> **Go vs Rust:** Section 1 matches Go's numbers case for case (same PCG cases, same paths taken, 0 failures). Section 2 differs only because the Rust fallback threshold is 32,768, the crossover example 12's Rust twin measured, not Go's 1,024. One generic `sort<T: RadixKey>` is monomorphised for `u64`/`i64`/`f64`, so floats need no separate key slice: 1 allocation where Go's `SortFloat64` makes 2. Strings move through the MSD buffer with `mem::take` (`S: AsRef<[u8]> + Default`) and are never cloned. The float oracle, `f64::total_cmp`, uses the same bit trick as our key (read in the source), so it is a less independent check than Go's by-cases comparator.
> Measured over several runs: the library itself runs at about the same speed in both languages (LSD ~5–10 ns/element). The opponent is what differs: it beats `sort_unstable` only 1.3–2.2x on the numeric workloads, against 6–9x over Go's `slices.Sort`, and on URLs it ties or loses (0.7–1.1x).

**Complexity:** Θ(p·n + p·2⁸) for the p byte passes actually needed (8 for random keys, 6 for one day of nanosecond timestamps), Θ(n + k) on the counting path, the stdlib's Θ(n log n) below n = 1,024 · space Θ(n): one buffer per call (two for `SortFloat64`, which needs a separate key slice) · stable: `SortByKey` yes. Proved: 0 failures in ~100,000 random and edge cases with our code forced on, and through the public API. Measured over several runs at n = 1M: about 6–9x faster than `slices.Sort` on the integer workloads, 4–9x on floats, and only 1.2–2.7x on URLs.

---

> Next lesson: [15 — Binary Search & Its Variants](../../15-binary-search.md)

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
