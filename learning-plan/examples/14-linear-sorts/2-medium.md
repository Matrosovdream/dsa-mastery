# Step 14 — Linear-Time Sorts · 🟡 Medium

Examples **7–11**: choosing the digit width, signed integers and floats through a bit transform,
LSD and MSD for strings, bucket sort and its assumption, and the problems where counting is the answer.

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

> ← Back to [🟢 easy](1-easy.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. Choosing the radix: bits per pass

`🟡 medium` · *Radix / passes vs histogram*

LSD radix sort on w-bit keys with b-bit digits makes ⌈w/b⌉ counting-sort passes, each reading n keys,
scanning 2ᵇ counters and moving n keys: Θ(⌈w/b⌉·(n + 2ᵇ)) in total. The digit width is the one knob,
and it trades passes against histogram size. This program counts the work for b = 1 … 16 on a million
random `uint64`, picks the b with the fewest operations, and then times the candidates — the count
picks one b and the clock another, and the gap is what a 512 KB histogram and 65,536 scattered write
targets cost. A digit width is a measurement, not a formula.

**Steps:**

1. Write one LSD sort with the digit width `b` as a parameter, counting reads, counter slots and moves per pass.
2. Run it for every b, check each result against `slices.Sort`, and compare the counts with ⌈w/b⌉·(2n + 2ᵇ).
3. Time b = 4, 8, 11, 16 on fresh copies of the input and read the *ns per counted op* column: on the machine below it stays near 0.5 ns up to b = 11 and doubles at b = 16.

```go
package main

import (
	"fmt"
	"math/bits"
	"math/rand/v2"
	"slices"
	"time"
)

// LSD radix sort on w-bit keys with b-bit digits makes ceil(w/b) passes. Each
// pass is one counting sort:
//
//	histogram: read n keys, bump count[digit]          n operations
//	prefix sum: turn 2^b counts into start positions   2^b operations
//	scatter:   move n keys to their start positions    n operations
//
// So the work is ceil(w/b) * (2n + 2^b). Small b: many passes. Large b: few
// passes, but a big histogram and 2^b different places to write to.

// stats counts the work, pass by pass. The counts are exact by construction:
// every pass adds exactly what its three loops do.
type stats struct {
	passes, reads, slots, moves int
}

func (s stats) work() int { return s.reads + s.slots + s.moves }

// key is the set of key types this sort accepts. The digit width b is a
// runtime value, so the histogram has to be a slice (Go has no const generics:
// an array length must be a constant, not a type parameter).
type key interface{ ~uint32 | ~uint64 }

// lsd sorts xs using buf (len(buf) == len(xs)) as the ping-pong buffer.
func lsd[K key](xs, buf []K, b int, st *stats) {
	w := bits.OnesCount64(uint64(^K(0))) // 32 or 64: the key width
	mask := K(1)<<b - 1
	count := make([]int, 1<<b) // allocated once, cleared every pass
	src, dst := xs, buf
	for shift := 0; shift < w; shift += b {
		clear(count)
		for _, x := range src { // histogram
			count[(x>>shift)&mask]++
		}
		sum := 0
		for d, c := range count { // exclusive prefix sum: count[d] = start of bucket d
			count[d] = sum
			sum += c
		}
		for _, x := range src { // scatter, in input order: this is what makes it stable
			d := (x >> shift) & mask
			dst[count[d]] = x
			count[d]++
		}
		st.passes++
		st.reads += len(src)
		st.slots += len(count)
		st.moves += len(src)
		src, dst = dst, src
	}
	if &src[0] != &xs[0] { // an odd number of passes left the result in buf
		copy(xs, src)
		st.moves += len(xs)
	}
}

func countedTable[K key](keys []K, widths []int, ref []K) {
	n := len(keys)
	w := bits.OnesCount64(uint64(^K(0)))
	fmt.Println("  counted work = reads + counter slots + moves; formula = passes*(2n + 2^b)")
	fmt.Println()
	fmt.Printf("  %3s  %6s  %9s  %14s  %16s  %14s\n",
		"b", "passes", "slots/pass", "counted work", "formula", "counter bytes")
	allMatch := true
	var odd []int
	for _, b := range widths {
		xs := slices.Clone(keys)
		buf := make([]K, n)
		var st stats
		lsd(xs, buf, b, &st)
		passes := (w + b - 1) / b
		formula := passes * (2*n + 1<<b)
		if passes%2 == 1 { // odd: the result ends in buf, copy back = n more moves
			formula += n
			odd = append(odd, b)
		}
		fmt.Printf("  %3d  %6d  %9d  %14d  %16d  %14d\n",
			b, st.passes, 1<<b, st.work(), formula, (1<<b)*8)
		if st.work() != formula {
			fmt.Println("  !! counted work differs from the formula")
		}
		allMatch = allMatch && slices.Equal(xs, ref)
	}
	for _, b := range odd {
		fmt.Printf("  (b = %d: %d passes is odd, so the result ends in buf: + n moves to copy it back)\n",
			b, (w+b-1)/b)
	}
	fmt.Printf("  every b above produced exactly slices.Sort's output: %v\n", allMatch)
}

func main() {
	const n = 1_000_000
	rng := rand.New(rand.NewPCG(1, 2))
	keys := make([]uint64, n)
	for i := range keys {
		keys[i] = rng.Uint64()
	}
	ref := slices.Clone(keys)
	slices.Sort(ref)

	fmt.Printf("1) counted work, n = %d random uint64 (w = 64)\n\n", n)
	countedTable(keys, []int{1, 2, 4, 8, 11, 16}, ref)

	// "On paper": minimise ceil(64/b)*(2n + 2^b) over every b from 1 to 24.
	bestB, bestW := 0, 0
	for b := 1; b <= 24; b++ {
		if wk := (64 + b - 1) / b * (2*n + 1<<b); bestB == 0 || wk < bestW {
			bestB, bestW = b, wk
		}
	}
	fmt.Printf("\n  fewest operations over b = 1..24: b = %d (%d passes, %d ops = %.2f per key)\n",
		bestB, (64+bestB-1)/bestB, bestW, float64(bestW)/n)

	// 2) The clock. Every run sorts a fresh copy of the same input (sorting an
	// already sorted slice would measure nothing), so a copy-only row is the
	// control to subtract mentally. Best of 5 runs per row.
	fmt.Printf("\n2) the clock, same n (best of 5 runs, each on a fresh copy)\n\n")
	work := make([]uint64, n)
	buf := make([]uint64, n)
	best := func(sort func(), check bool) float64 {
		bestNs := int64(-1)
		for range 5 {
			t := time.Now()
			copy(work, keys)
			sort()
			if ns := time.Since(t).Nanoseconds(); bestNs < 0 || ns < bestNs {
				bestNs = ns
			}
		}
		if check && !slices.Equal(work, ref) {
			panic("not sorted")
		}
		return float64(bestNs) / n
	}
	copyOnly := best(func() {}, false) // the copy alone
	fmt.Printf("  %-14s  %10s  %15s\n", "", "ns/element", "ns per counted op")
	fmt.Printf("  %-14s  %10.2f  %15s\n", "copy only", copyOnly, "-")
	fastB, fastNs := 0, 0.0
	nsOf := map[int]float64{}
	for _, b := range []int{4, 8, 11, 16} {
		var st stats
		ns := best(func() { st = stats{}; lsd(work, buf, b, &st) }, true)
		nsOf[b] = ns
		fmt.Printf("  %-14s  %10.2f  %15.3f\n", fmt.Sprintf("LSD b = %d", b), ns,
			ns*n/float64(st.work()))
		if fastB == 0 || ns < fastNs {
			fastB, fastNs = b, ns
		}
	}
	pdq := best(func() { slices.Sort(work) }, true)
	fmt.Printf("  %-14s  %10.2f  %15s\n", "slices.Sort", pdq, "-")

	fmt.Printf("\n  fastest b on the clock this run: %d", fastB)
	if fastB == bestB {
		fmt.Printf(" (the same as the count's choice)\n")
	} else {
		fmt.Printf(" -- not b = %d, the count's choice\n", bestB)
	}
	fmt.Printf("  b = 16 does %.2fx fewer counted ops than b = 8 and takes %.2fx as long\n",
		float64(8*(2*n+256))/float64(4*(2*n+65536)), nsOf[16]/nsOf[8])

	// 3) Narrower keys need fewer passes for the same b.
	fmt.Printf("\n3) the same table for n = %d random uint32 (w = 32)\n\n", n)
	k32 := make([]uint32, n)
	for i := range k32 {
		k32[i] = rng.Uint32()
	}
	ref32 := slices.Clone(k32)
	slices.Sort(ref32)
	countedTable(k32, []int{4, 8, 11, 16}, ref32)
}
```

**Sample output:**

```
1) counted work, n = 1000000 random uint64 (w = 64)

  counted work = reads + counter slots + moves; formula = passes*(2n + 2^b)

    b  passes  slots/pass    counted work           formula   counter bytes
    1      64          2       128000128         128000128              16
    2      32          4        64000128          64000128              32
    4      16         16        32000256          32000256             128
    8       8        256        16002048          16002048            2048
   11       6       2048        12012288          12012288           16384
   16       4      65536         8262144           8262144          524288
  every b above produced exactly slices.Sort's output: true

  fewest operations over b = 1..24: b = 16 (4 passes, 8262144 ops = 8.26 per key)

2) the clock, same n (best of 5 runs, each on a fresh copy)

                  ns/element  ns per counted op
  copy only             0.39                -
  LSD b = 4            17.65            0.551
  LSD b = 8             8.28            0.517
  LSD b = 11            6.68            0.556
  LSD b = 16            8.54            1.034
  slices.Sort          58.75                -

  fastest b on the clock this run: 11 -- not b = 16, the count's choice
  b = 16 does 1.94x fewer counted ops than b = 8 and takes 1.03x as long

3) the same table for n = 1000000 random uint32 (w = 32)

  counted work = reads + counter slots + moves; formula = passes*(2n + 2^b)

    b  passes  slots/pass    counted work           formula   counter bytes
    4       8         16        16000128          16000128             128
    8       4        256         8001024           8001024            2048
   11       3       2048         7006144           7006144           16384
   16       2      65536         4131072           4131072          524288
  (b = 11: 3 passes is odd, so the result ends in buf: + n moves to copy it back)
  every b above produced exactly slices.Sort's output: true
```

**Rust:**

```rust
use std::time::Instant;

// LSD radix sort on w-bit keys with b-bit digits makes ceil(w/b) passes. Each
// pass is one counting sort:
//
//   histogram:  read n keys, bump count[digit]          n operations
//   prefix sum: turn 2^b counts into start positions    2^b operations
//   scatter:    move n keys to their start positions    n operations
//
// So the work is ceil(w/b) * (2n + 2^b). Small b: many passes. Large b: few
// passes, but a big histogram and 2^b different places to write to.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state), so that
// seed (1, 2) gives exactly the keys the Go program sorts.
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

    /// like Go's Rand.Uint32: the top 32 bits of one Uint64
    fn next_u32(&mut self) -> u32 {
        (self.next_u64() >> 32) as u32
    }
}

// The work, counted pass by pass. Exact by construction: every pass adds
// exactly what its three loops do.
#[derive(Default)]
struct Stats {
    passes: usize,
    reads: usize,
    slots: usize,
    moves: usize,
}

impl Stats {
    fn work(&self) -> usize {
        self.reads + self.slots + self.moves
    }
}

// The key types this sort accepts: u32 and u64.
trait Key: Copy + Ord {
    const BITS: u32;
    fn shifted(self, shift: u32) -> usize;
}
impl Key for u64 {
    const BITS: u32 = 64;
    fn shifted(self, shift: u32) -> usize {
        (self >> shift) as usize
    }
}
impl Key for u32 {
    const BITS: u32 = 32;
    fn shifted(self, shift: u32) -> usize {
        (self >> shift) as usize
    }
}

// lsd sorts xs using buf (same length) as the ping-pong buffer. The radix R
// = 2^b is a const generic, so the histogram is a fixed-size STACK array
// [usize; R] -- Go has no const generics and needs a slice here. (Stable Rust
// does not accept `[usize; 1 << B]` for a `const B: u32` parameter: arithmetic
// on a const parameter in a type is still unstable. So the parameter is R
// itself, and b = R.trailing_zeros().) The mask `& (R - 1)` is a constant too,
// so the compiler can see every count[...] index is in bounds.
fn lsd<K: Key, const R: usize>(xs: &mut [K], buf: &mut [K], st: &mut Stats) {
    let b = R.trailing_zeros();
    let mut count = [0usize; R];
    let mut in_xs = true; // where the current data lives
    let mut shift = 0;
    while shift < K::BITS {
        // Two separate &mut parameters, so one can be read while the other is
        // written: the borrow checker sees they cannot alias.
        let (src, dst): (&[K], &mut [K]) = if in_xs { (xs, buf) } else { (buf, xs) };
        count.fill(0);
        for &x in src.iter() {
            count[x.shifted(shift) & (R - 1)] += 1; // histogram
        }
        let mut sum = 0;
        for c in count.iter_mut() {
            // exclusive prefix sum: count[d] = start of bucket d
            let v = *c;
            *c = sum;
            sum += v;
        }
        for &x in src.iter() {
            // scatter, in input order: this is what makes it stable
            let d = x.shifted(shift) & (R - 1);
            dst[count[d]] = x;
            count[d] += 1;
        }
        st.passes += 1;
        st.reads += src.len();
        st.slots += R;
        st.moves += src.len();
        in_xs = !in_xs;
        shift += b;
    }
    if !in_xs {
        // an odd number of passes left the result in buf
        xs.copy_from_slice(buf);
        st.moves += xs.len();
    }
}

// The run-time b picks one monomorphised copy per width.
fn lsd_b<K: Key>(b: u32, xs: &mut [K], buf: &mut [K], st: &mut Stats) {
    match b {
        1 => lsd::<K, 2>(xs, buf, st),
        2 => lsd::<K, 4>(xs, buf, st),
        4 => lsd::<K, 16>(xs, buf, st),
        8 => lsd::<K, 256>(xs, buf, st),
        11 => lsd::<K, 2048>(xs, buf, st),
        16 => lsd::<K, 65536>(xs, buf, st),
        _ => unreachable!("no instance for b = {b}"),
    }
}

fn counted_table<K: Key + Default>(keys: &[K], widths: &[u32], reference: &[K]) {
    let n = keys.len();
    let w = K::BITS as usize;
    println!("  counted work = reads + counter slots + moves; formula = passes*(2n + 2^b)");
    println!();
    println!(
        "  {:>3}  {:>6}  {:>9}  {:>14}  {:>16}  {:>14}",
        "b", "passes", "slots/pass", "counted work", "formula", "counter bytes"
    );
    let mut all_match = true;
    let mut odd = Vec::new();
    for &b in widths {
        let mut xs = keys.to_vec();
        let mut buf = vec![K::default(); n];
        let mut st = Stats::default();
        lsd_b(b, &mut xs, &mut buf, &mut st);
        let bu = b as usize;
        let passes = w.div_ceil(bu);
        let mut formula = passes * (2 * n + (1 << b));
        if passes % 2 == 1 {
            // odd: the result ends in buf, copy back = n more moves
            formula += n;
            odd.push(b);
        }
        println!(
            "  {:>3}  {:>6}  {:>9}  {:>14}  {:>16}  {:>14}",
            b,
            st.passes,
            1usize << b,
            st.work(),
            formula,
            (1usize << b) * size_of::<usize>()
        );
        if st.work() != formula {
            println!("  !! counted work differs from the formula");
        }
        all_match &= xs == reference;
    }
    for b in odd {
        println!(
            "  (b = {b}: {} passes is odd, so the result ends in buf: + n moves to copy it back)",
            w.div_ceil(b as usize)
        );
    }
    println!("  every b above produced exactly sort_unstable's output: {all_match}");
}

fn main() {
    const N: usize = 1_000_000;
    let mut rng = Pcg::new(1, 2);
    let keys: Vec<u64> = (0..N).map(|_| rng.next_u64()).collect();
    let mut reference = keys.clone();
    reference.sort_unstable();

    println!("1) counted work, n = {N} random u64 (w = 64)\n");
    counted_table(&keys, &[1, 2, 4, 8, 11, 16], &reference);

    // "On paper": minimise ceil(64/b)*(2n + 2^b) over every b from 1 to 24.
    let (best_b, best_w) = (1..=24usize)
        .map(|b| (b, 64usize.div_ceil(b) * (2 * N + (1 << b))))
        .min_by_key(|&(b, wk)| (wk, b)) // ties: the smaller b, as Go's strict `<` keeps the first
        .unwrap();
    println!(
        "\n  fewest operations over b = 1..24: b = {best_b} ({} passes, {best_w} ops = {:.2} per key)",
        64usize.div_ceil(best_b),
        best_w as f64 / N as f64
    );

    // 2) The clock. Every run sorts a fresh copy of the same input (sorting an
    // already sorted slice would measure nothing), so a copy-only row is the
    // control. Best of 5 runs per row.
    println!("\n2) the clock, same n (best of 5 runs, each on a fresh copy)\n");
    let mut work = vec![0u64; N];
    let mut buf = vec![0u64; N];
    let mut best = |sort: &mut dyn FnMut(&mut Vec<u64>, &mut Vec<u64>), check: bool| -> f64 {
        let mut best_ns = u128::MAX;
        for _ in 0..5 {
            let t = Instant::now();
            work.copy_from_slice(&keys);
            sort(&mut work, &mut buf);
            best_ns = best_ns.min(t.elapsed().as_nanos());
        }
        assert!(!check || work == reference, "not sorted");
        best_ns as f64 / N as f64
    };
    let copy_only = best(&mut |w, _| _ = std::hint::black_box(w), false);
    println!(
        "  {:<14}  {:>10}  {:>15}",
        "", "ns/element", "ns per counted op"
    );
    println!("  {:<14}  {:>10.2}  {:>15}", "copy only", copy_only, "-");
    let mut fastest: Option<(u32, f64)> = None;
    let mut ns_of = [0.0f64; 17];
    for b in [4u32, 8, 11, 16] {
        let mut st = Stats::default();
        let ns = best(
            &mut |w, bf| {
                st = Stats::default();
                lsd_b(b, w, bf, &mut st);
            },
            true,
        );
        ns_of[b as usize] = ns;
        println!(
            "  {:<14}  {:>10.2}  {:>15.3}",
            format!("LSD b = {b}"),
            ns,
            ns * N as f64 / st.work() as f64
        );
        if fastest.is_none_or(|(_, f)| ns < f) {
            fastest = Some((b, ns));
        }
    }
    let ipn = best(&mut |w, _| w.sort_unstable(), true);
    println!("  {:<14}  {:>10.2}  {:>15}", "sort_unstable", ipn, "-");

    let (fast_b, _) = fastest.unwrap();
    print!("\n  fastest b on the clock this run: {fast_b}");
    if fast_b as usize == best_b {
        println!(" (the same as the count's choice)");
    } else {
        println!(" -- not b = {best_b}, the count's choice");
    }
    println!(
        "  b = 16 does {:.2}x fewer counted ops than b = 8 and takes {:.2}x as long",
        (8 * (2 * N + 256)) as f64 / (4 * (2 * N + 65536)) as f64,
        ns_of[16] / ns_of[8]
    );

    // 3) Narrower keys need fewer passes for the same b.
    println!("\n3) the same table for n = {N} random u32 (w = 32)\n");
    let k32: Vec<u32> = (0..N).map(|_| rng.next_u32()).collect();
    let mut ref32 = k32.clone();
    ref32.sort_unstable();
    counted_table(&k32, &[4, 8, 11, 16], &ref32);
}
```

**Rust sample output:**

```
1) counted work, n = 1000000 random u64 (w = 64)

  counted work = reads + counter slots + moves; formula = passes*(2n + 2^b)

    b  passes  slots/pass    counted work           formula   counter bytes
    1      64          2       128000128         128000128              16
    2      32          4        64000128          64000128              32
    4      16         16        32000256          32000256             128
    8       8        256        16002048          16002048            2048
   11       6       2048        12012288          12012288           16384
   16       4      65536         8262144           8262144          524288
  every b above produced exactly sort_unstable's output: true

  fewest operations over b = 1..24: b = 16 (4 passes, 8262144 ops = 8.26 per key)

2) the clock, same n (best of 5 runs, each on a fresh copy)

                  ns/element  ns per counted op
  copy only             0.11                -
  LSD b = 4            18.01            0.563
  LSD b = 8             6.44            0.403
  LSD b = 11            5.14            0.428
  LSD b = 16            7.91            0.957
  sort_unstable        10.16                -

  fastest b on the clock this run: 11 -- not b = 16, the count's choice
  b = 16 does 1.94x fewer counted ops than b = 8 and takes 1.23x as long

3) the same table for n = 1000000 random u32 (w = 32)

  counted work = reads + counter slots + moves; formula = passes*(2n + 2^b)

    b  passes  slots/pass    counted work           formula   counter bytes
    4       8         16        16000128          16000128             128
    8       4        256         8001024           8001024            2048
   11       3       2048         7006144           7006144           16384
   16       2      65536         4131072           4131072          524288
  (b = 11: 3 passes is odd, so the result ends in buf: + n moves to copy it back)
  every b above produced exactly sort_unstable's output: true
```

> **Go vs Rust:** Sections 1 and 3 (every count, the count's choice b = 16) are identical to Go's — same PCG keys, same algorithm; only the labels differ. The clock agreed on the shape in both languages (b = 11 fastest in every run, b = 16 about twice the ns per counted op), but not on the reference row: `sort_unstable` (ipnsort) took ≈ 10 ns/element where Go's `slices.Sort` (pdqsort) took ≈ 57 on the same keys, so in Rust the best LSD wins by about 2x, not about 9x — why ipnsort is that much faster was not measured.
> The histogram is a stack array `[usize; R]` through a const generic `R`; the plan's `[usize; 1 << B]` is rejected on stable Rust ("generic parameters may not be used in const operations"), so the radix itself is the parameter.

**Complexity:** time Θ(⌈w/b⌉·(n + 2ᵇ)) · space Θ(n + 2ᵇ). At n = 10⁶ the count's best is b = 16 (4 passes, 8.26 ops per key), but b = 11 was fastest on the clock in every run here (≈ 6–7.5 ns/element, b = 8 and b = 16 ≈ 8–9, `slices.Sort` ≈ 57–60): b = 16 pays about twice as much per counted operation. On paper the best b is the fewest passes whose 2ᵇ stays below n; in practice it is whatever keeps the histogram in cache.

---

## 8. Signed keys and floats: flip the bits

`🟡 medium` · *Key transforms*

A radix sort orders unsigned bit patterns and nothing else, so every other key type goes through an
**order-preserving** map to `uint64`: sort the mapped keys, map back. For `int64` the map is one XOR
— flip the sign bit, so two's-complement negatives stop sorting above the positives. For `float64` it
is two: flip *all* bits of a negative (larger magnitude must sort lower), flip only the sign bit of a
non-negative. That is the IEEE 754 totalOrder, −NaN < −∞ < … < −0 < +0 < … < +∞ < +NaN — and it is
*not* the order `slices.Sort` uses, which puts every NaN first and treats −0 and +0 as equal. The
transform is Θ(1) per key, so the sort stays Θ(d·(n + 2ᵇ)).

**Steps:**

1. Radix-sort nine `int64` keys (with `MinInt64` and `MaxInt64`) on their raw bits, then on `x ^ 1<<63`, and print both keys in hex.
2. Build twelve `float64` keys with ±0, ±∞, ±NaN and the ± smallest subnormal, print raw and transformed bits, and check the transform round-trips.
3. Compare the radix order with `slices.Sort` and with a `SortFunc` on the key bits, and name the positions where they disagree.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"slices"
	"strings"
)

// A radix sort orders unsigned bit patterns, nothing else. To sort anything
// else, map each key to a uint64 by a transform that PRESERVES ORDER
// (a < b  <=>  key(a) < key(b)), sort the uint64s, and map back.

// radix sorts uint64 keys: LSD, 8 passes of 8 bits (lesson example 5).
func radix(xs []uint64) {
	buf := make([]uint64, len(xs))
	src, dst := xs, buf
	for shift := 0; shift < 64; shift += 8 {
		var count [256]int
		for _, x := range src {
			count[byte(x>>shift)]++
		}
		sum := 0
		for d, c := range count {
			count[d] = sum
			sum += c
		}
		for _, x := range src {
			d := byte(x >> shift)
			dst[count[d]] = x
			count[d]++
		}
		src, dst = dst, src
	}
	// 8 passes is even: the result is back in xs.
}

// int64: two's complement puts the sign in the top bit, so as unsigned
// numbers every negative is LARGER than every positive. Flipping the sign bit
// fixes exactly that: MinInt64 -> 0, -1 -> 0x7fff..., 0 -> 0x8000...
func intKey(x int64) uint64     { return uint64(x) ^ 1<<63 }
func intFromKey(k uint64) int64 { return int64(k ^ 1<<63) }

// float64: sign | exponent | mantissa. For x >= 0 the bits already sort like
// unsigned ints, so set the sign bit to put them above all negatives. For
// x < 0 a larger magnitude has larger bits but must sort LOWER: flip them all.
// This is the IEEE 754 totalOrder: -NaN < -Inf < ... < -0 < +0 < ... < +Inf < +NaN.
func floatKey(f float64) uint64 {
	b := math.Float64bits(f)
	if b>>63 == 1 {
		return ^b
	}
	return b | 1<<63
}

func floatFromKey(k uint64) float64 {
	if k>>63 == 1 {
		return math.Float64frombits(k &^ (1 << 63))
	}
	return math.Float64frombits(^k)
}

// sortBy is the whole recipe: transform, radix sort, transform back.
func sortBy[T any](xs []T, key func(T) uint64, back func(uint64) T) []T {
	ks := make([]uint64, len(xs))
	for i, x := range xs {
		ks[i] = key(x)
	}
	radix(ks)
	out := make([]T, len(xs))
	for i, k := range ks {
		out[i] = back(k)
	}
	return out
}

// show prints a float so that the sign of zero and of NaN is visible:
// Go's %v prints "0" for -0 and "NaN" for both NaNs.
func show(f float64) string {
	switch {
	case math.IsNaN(f) && math.Signbit(f):
		return "-NaN"
	case math.IsNaN(f):
		return "NaN"
	case f == 0 && math.Signbit(f):
		return "-0"
	}
	return fmt.Sprintf("%v", f)
}

func showAll(xs []float64) string {
	parts := make([]string, len(xs))
	for i, x := range xs {
		parts[i] = show(x)
	}
	return "[" + strings.Join(parts, " ") + "]"
}

// sameBits compares floats by bit pattern: == would say NaN != NaN and -0 == 0.
func sameBits(a, b []float64) bool {
	return slices.EqualFunc(a, b, func(x, y float64) bool {
		return math.Float64bits(x) == math.Float64bits(y)
	})
}

func main() {
	fmt.Println("1) int64")
	ints := []int64{5, -1, math.MinInt64, 0, math.MaxInt64, -42, 42, 1, -2}
	fmt.Println("  input:             ", ints)
	raw := sortBy(ints, func(x int64) uint64 { return uint64(x) },
		func(k uint64) int64 { return int64(k) })
	fmt.Println("  radix on raw bits: ", raw, " <- negatives after positives")
	fmt.Println()
	fmt.Printf("  %20s  %16s  %16s\n", "x", "raw bits", "x ^ 1<<63")
	for _, x := range ints {
		fmt.Printf("  %20d  %016x  %016x\n", x, uint64(x), intKey(x))
	}
	fixed := sortBy(ints, intKey, intFromKey)
	want := slices.Clone(ints)
	slices.Sort(want)
	fmt.Println()
	fmt.Println("  radix on x ^ 1<<63:", fixed)
	fmt.Println("  matches slices.Sort:", slices.Equal(fixed, want))

	fmt.Println()
	fmt.Println("2) float64")
	// Two traps when WRITING these keys in Go:
	//  - the constant -0.0 is +0: constant arithmetic is exact, and exact
	//    zero has no sign. A negative zero must be made at run time.
	//  - math.NaN() is 0x7ff8000000000001 (math/bits.go: uvnan). This program
	//    spells its NaNs out bit by bit so the Rust twin uses the same ones.
	negZero := math.Copysign(0, -1)
	fmt.Printf("  the constant -0.0 has its sign bit set: %v; math.Copysign(0, -1) does: %v\n",
		math.Signbit(-0.0), math.Signbit(negZero))
	nan := math.Float64frombits(0x7ff8000000000000)
	negNaN := math.Float64frombits(0xfff8000000000000)
	floats := []float64{3.14, -2.5, 0, negZero, math.Inf(1), math.Inf(-1),
		5e-324, -5e-324, nan, negNaN, 1, -1}
	fmt.Println()
	fmt.Printf("  %8s  %16s  %16s\n", "x", "raw bits", "key bits")
	for _, x := range floats {
		fmt.Printf("  %8s  %016x  %016x\n", show(x), math.Float64bits(x), floatKey(x))
	}
	// The transform must be a bijection, or sorting would lose keys.
	roundTrip := true
	for _, x := range floats {
		roundTrip = roundTrip && math.Float64bits(floatFromKey(floatKey(x))) == math.Float64bits(x)
	}
	fmt.Println("  key -> float -> key round-trips every value bit for bit:", roundTrip)

	rad := sortBy(floats, floatKey, floatFromKey)
	fmt.Println()
	fmt.Println("  radix (totalOrder):          ", showAll(rad))

	// Go's stdlib: slices.Sort uses cmp.Less, which puts NaN below everything
	// and has -0 == 0 (cmp/cmp.go). With n = 12 pdqsort is pure insertion sort
	// (maxInsertion = 12, slices/zsortordered.go), which is stable: elements
	// that compare equal stay in INPUT order. So this line is deterministic.
	std := slices.Clone(floats)
	slices.Sort(std)
	fmt.Println("  slices.Sort:                 ", showAll(std))
	fmt.Println("  slices.Sort agrees with radix, bit for bit:", sameBits(std, rad))

	// Go has no total-order float comparator in the stdlib; the transform
	// itself is one.
	tot := slices.Clone(floats)
	slices.SortFunc(tot, func(a, b float64) int { return cmp.Compare(floatKey(a), floatKey(b)) })
	fmt.Println("  SortFunc on the key bits:    ", showAll(tot))
	fmt.Println("  key-bits SortFunc agrees with radix, bit for bit:", sameBits(tot, rad))

	// Name the disagreements from the two outputs, not from memory.
	pos := func(xs []float64, bitsOf uint64) int {
		return slices.IndexFunc(xs, func(x float64) bool { return math.Float64bits(x) == bitsOf })
	}
	fmt.Println()
	fmt.Println("  where they disagree:")
	fmt.Printf("    NaN:  radix puts -NaN at %d and NaN at %d (by sign); slices.Sort puts them at %d and %d\n",
		pos(rad, math.Float64bits(negNaN)), pos(rad, math.Float64bits(nan)),
		pos(std, math.Float64bits(negNaN)), pos(std, math.Float64bits(nan)))
	fmt.Printf("    zero: radix puts -0 at %d and 0 at %d; slices.Sort puts them at %d and %d\n",
		pos(rad, math.Float64bits(negZero)), pos(rad, 0),
		pos(std, math.Float64bits(negZero)), pos(std, 0))
	first := "0"
	if pos(floats, math.Float64bits(negZero)) < pos(floats, 0) {
		first = "-0"
	}
	fmt.Printf("          (to slices.Sort -0 == 0, so they keep their input order, and %s came first)\n", first)
	notNaN := func(xs []float64) []float64 {
		return slices.DeleteFunc(slices.Clone(xs), math.IsNaN)
	}
	r, t := notNaN(rad), notNaN(std)
	var differ []string
	for i := range r {
		if math.Float64bits(r[i]) != math.Float64bits(t[i]) {
			differ = append(differ, fmt.Sprintf("%d (%s vs %s)", i, show(r[i]), show(t[i])))
		}
	}
	fmt.Printf("    drop the NaNs from both: %d of %d positions match; the rest: %s\n",
		len(r)-len(differ), len(r), strings.Join(differ, ", "))
}
```

**Output:**

```
1) int64
  input:              [5 -1 -9223372036854775808 0 9223372036854775807 -42 42 1 -2]
  radix on raw bits:  [0 1 5 42 9223372036854775807 -9223372036854775808 -42 -2 -1]  <- negatives after positives

                     x          raw bits         x ^ 1<<63
                     5  0000000000000005  8000000000000005
                    -1  ffffffffffffffff  7fffffffffffffff
  -9223372036854775808  8000000000000000  0000000000000000
                     0  0000000000000000  8000000000000000
   9223372036854775807  7fffffffffffffff  ffffffffffffffff
                   -42  ffffffffffffffd6  7fffffffffffffd6
                    42  000000000000002a  800000000000002a
                     1  0000000000000001  8000000000000001
                    -2  fffffffffffffffe  7ffffffffffffffe

  radix on x ^ 1<<63: [-9223372036854775808 -42 -2 -1 0 1 5 42 9223372036854775807]
  matches slices.Sort: true

2) float64
  the constant -0.0 has its sign bit set: false; math.Copysign(0, -1) does: true

         x          raw bits          key bits
      3.14  40091eb851eb851f  c0091eb851eb851f
      -2.5  c004000000000000  3ffbffffffffffff
         0  0000000000000000  8000000000000000
        -0  8000000000000000  7fffffffffffffff
      +Inf  7ff0000000000000  fff0000000000000
      -Inf  fff0000000000000  000fffffffffffff
    5e-324  0000000000000001  8000000000000001
   -5e-324  8000000000000001  7ffffffffffffffe
       NaN  7ff8000000000000  fff8000000000000
      -NaN  fff8000000000000  0007ffffffffffff
         1  3ff0000000000000  bff0000000000000
        -1  bff0000000000000  400fffffffffffff
  key -> float -> key round-trips every value bit for bit: true

  radix (totalOrder):           [-NaN -Inf -2.5 -1 -5e-324 -0 0 5e-324 1 3.14 +Inf NaN]
  slices.Sort:                  [NaN -NaN -Inf -2.5 -1 -5e-324 0 -0 5e-324 1 3.14 +Inf]
  slices.Sort agrees with radix, bit for bit: false
  SortFunc on the key bits:     [-NaN -Inf -2.5 -1 -5e-324 -0 0 5e-324 1 3.14 +Inf NaN]
  key-bits SortFunc agrees with radix, bit for bit: true

  where they disagree:
    NaN:  radix puts -NaN at 0 and NaN at 11 (by sign); slices.Sort puts them at 1 and 0
    zero: radix puts -0 at 5 and 0 at 6; slices.Sort puts them at 7 and 6
          (to slices.Sort -0 == 0, so they keep their input order, and 0 came first)
    drop the NaNs from both: 8 of 10 positions match; the rest: 4 (-0 vs 0), 5 (0 vs -0)
```

**Rust:**

```rust
use std::cmp::Ordering;
use std::fmt::Display;

// A radix sort orders unsigned bit patterns, nothing else. To sort anything
// else, map each key to a u64 by a transform that PRESERVES ORDER
// (a < b  <=>  key(a) < key(b)), sort the u64s, and map back.

/// LSD, 8 passes of 8 bits (lesson example 5).
fn radix(xs: &mut [u64]) {
    let mut buf = vec![0u64; xs.len()];
    let (mut src, mut dst): (&mut [u64], &mut [u64]) = (xs, &mut buf);
    for shift in (0..64).step_by(8) {
        let mut count = [0usize; 256];
        for &x in src.iter() {
            count[usize::from((x >> shift) as u8)] += 1;
        }
        let mut sum = 0;
        for c in count.iter_mut() {
            let v = *c;
            *c = sum;
            sum += v;
        }
        for &x in src.iter() {
            let d = usize::from((x >> shift) as u8);
            dst[count[d]] = x;
            count[d] += 1;
        }
        std::mem::swap(&mut src, &mut dst);
    }
    // 8 passes is even: the result is back in xs.
}

// i64: two's complement puts the sign in the top bit, so as unsigned numbers
// every negative is LARGER than every positive. Flipping the sign bit fixes
// exactly that: MIN -> 0, -1 -> 0x7fff..., 0 -> 0x8000...
// (`x as u64` reinterprets the bits, like Go's uint64(x).)
fn int_key(x: i64) -> u64 {
    x as u64 ^ 1 << 63
}
fn int_from_key(k: u64) -> i64 {
    (k ^ 1 << 63) as i64
}

// f64: sign | exponent | mantissa. For x >= 0 the bits already sort like
// unsigned ints, so set the sign bit to put them above all negatives. For
// x < 0 a larger magnitude has larger bits but must sort LOWER: flip them all.
// This is the IEEE 754 totalOrder: -NaN < -Inf < ... < -0 < +0 < ... < +Inf < +NaN.
fn float_key(f: f64) -> u64 {
    let b = f.to_bits();
    if b >> 63 == 1 { !b } else { b | 1 << 63 }
}
fn float_from_key(k: u64) -> f64 {
    if k >> 63 == 1 {
        f64::from_bits(k & !(1 << 63))
    } else {
        f64::from_bits(!k)
    }
}

/// The whole recipe: transform, radix sort, transform back.
fn sort_by<T: Copy>(xs: &[T], key: impl Fn(T) -> u64, back: impl Fn(u64) -> T) -> Vec<T> {
    let mut ks: Vec<u64> = xs.iter().map(|&x| key(x)).collect();
    radix(&mut ks);
    ks.into_iter().map(back).collect()
}

/// Go's `[a b c]` slice format.
fn go_slice<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

/// Print a float so that the sign of zero and of NaN is visible, in Go's
/// spelling (+Inf, 5e-324) so the two programs' tables line up. Rust's `{}`
/// prints "-0" for negative zero already, but "NaN" for both NaNs, "inf",
/// and 5e-324 as 0.000...5 with 323 zeros.
fn show(f: f64) -> String {
    if f.is_nan() {
        return if f.is_sign_negative() { "-NaN" } else { "NaN" }.into();
    }
    if f.is_infinite() {
        return if f > 0.0 { "+Inf" } else { "-Inf" }.into();
    }
    if f == 0.0 {
        return if f.is_sign_negative() { "-0" } else { "0" }.into();
    }
    if f.abs() < 1e-4 {
        return format!("{f:e}"); // Go's %v switches to an exponent here too
    }
    f.to_string()
}

fn show_all(xs: &[f64]) -> String {
    let parts: Vec<String> = xs.iter().map(|&x| show(x)).collect();
    format!("[{}]", parts.join(" "))
}

/// Compare floats by bit pattern: == would say NaN != NaN and -0 == 0.
fn same_bits(a: &[f64], b: &[f64]) -> bool {
    a.len() == b.len() && a.iter().zip(b).all(|(x, y)| x.to_bits() == y.to_bits())
}

/// Go's cmp.Less, the rule slices.Sort uses (cmp/cmp.go):
/// (isNaN(x) && !isNaN(y)) || x < y.
fn go_less(x: f64, y: f64) -> bool {
    (x.is_nan() && !y.is_nan()) || x < y
}

fn main() {
    println!("1) i64");
    let ints = [5i64, -1, i64::MIN, 0, i64::MAX, -42, 42, 1, -2];
    println!("  input:              {}", go_slice(&ints));
    let raw = sort_by(&ints, |x| x as u64, |k| k as i64);
    println!(
        "  radix on raw bits:  {}  <- negatives after positives",
        go_slice(&raw)
    );
    println!();
    println!("  {:>20}  {:>16}  {:>16}", "x", "raw bits", "x ^ 1<<63");
    for &x in &ints {
        println!("  {x:>20}  {:016x}  {:016x}", x as u64, int_key(x));
    }
    let fixed = sort_by(&ints, int_key, int_from_key);
    let mut want = ints.to_vec();
    want.sort_unstable();
    println!();
    println!("  radix on x ^ 1<<63: {}", go_slice(&fixed));
    println!("  matches sort_unstable: {}", fixed == want);

    println!();
    println!("2) f64");
    // Rust has neither of Go's traps here: the literal -0.0 IS negative zero
    // (no exact-constant arithmetic), and the NaNs are spelled out bit by bit
    // anyway -- f64::NAN's bit pattern is documented as unspecified.
    let neg_zero = -0.0_f64;
    println!(
        "  the literal -0.0 has its sign bit set: {}; 0.0_f64.copysign(-1.0) does: {}",
        neg_zero.is_sign_negative(),
        0.0_f64.copysign(-1.0).is_sign_negative()
    );
    let nan = f64::from_bits(0x7ff8000000000000);
    let neg_nan = f64::from_bits(0xfff8000000000000);
    // 3.14 is a sample key, not an attempt at pi: the same twelve keys as Go.
    #[allow(clippy::approx_constant)]
    let floats = [
        3.14,
        -2.5,
        0.0,
        neg_zero,
        f64::INFINITY,
        f64::NEG_INFINITY,
        5e-324,
        -5e-324,
        nan,
        neg_nan,
        1.0,
        -1.0,
    ];
    println!();
    println!("  {:>8}  {:>16}  {:>16}", "x", "raw bits", "key bits");
    for &x in &floats {
        println!(
            "  {:>8}  {:016x}  {:016x}",
            show(x),
            x.to_bits(),
            float_key(x)
        );
    }
    // The transform must be a bijection, or sorting would lose keys.
    let round_trip = floats
        .iter()
        .all(|&x| float_from_key(float_key(x)).to_bits() == x.to_bits());
    println!("  key -> float -> key round-trips every value bit for bit: {round_trip}");

    let rad = sort_by(&floats, float_key, float_from_key);
    println!();
    println!("  radix (totalOrder):           {}", show_all(&rad));

    // Go's order, for comparison: cmp.Less under a STABLE sort, which is
    // exactly what Go's slices.Sort does at n <= 12 (pure insertion sort).
    // cmp.Less is a valid strict weak order (all NaNs one class at the bottom,
    // -0 == 0), so sort_by accepts it.
    let mut go = floats.to_vec();
    go.sort_by(|&a, &b| {
        if go_less(a, b) {
            Ordering::Less
        } else if go_less(b, a) {
            Ordering::Greater
        } else {
            Ordering::Equal
        }
    });
    println!("  Go's cmp.Less, stable sort:   {}", show_all(&go));
    println!(
        "  Go's rule agrees with radix, bit for bit: {}",
        same_bits(&go, &rad)
    );

    // Rust's stdlib: f64 is not Ord (NaN), so `v.sort()` does not compile.
    // You must pick an order, and the std one is total_cmp -- whose source
    // (core/src/num/f64.rs) is this same transform: flip all non-sign bits of
    // negatives, then compare as i64.
    let mut tot = floats.to_vec();
    tot.sort_unstable_by(f64::total_cmp);
    println!("  sort_unstable_by(total_cmp):  {}", show_all(&tot));
    println!(
        "  total_cmp agrees with radix, bit for bit: {}",
        same_bits(&tot, &rad)
    );
    let pairs = floats
        .iter()
        .flat_map(|&a| floats.iter().map(move |&b| (a, b)));
    let all = pairs
        .clone()
        .all(|(a, b)| a.total_cmp(&b) == float_key(a).cmp(&float_key(b)));
    println!(
        "  total_cmp(a, b) == key(a).cmp(key(b)) for all {} pairs: {all}",
        pairs.count()
    );

    // Name the disagreements from the two outputs, not from memory.
    let pos = |xs: &[f64], bits: u64| xs.iter().position(|x| x.to_bits() == bits).unwrap();
    println!();
    println!("  where radix and Go's rule disagree:");
    println!(
        "    NaN:  radix puts -NaN at {} and NaN at {} (by sign); Go's rule puts them at {} and {}",
        pos(&rad, neg_nan.to_bits()),
        pos(&rad, nan.to_bits()),
        pos(&go, neg_nan.to_bits()),
        pos(&go, nan.to_bits())
    );
    println!(
        "    zero: radix puts -0 at {} and 0 at {}; Go's rule puts them at {} and {}",
        pos(&rad, neg_zero.to_bits()),
        pos(&rad, 0),
        pos(&go, neg_zero.to_bits()),
        pos(&go, 0)
    );
    let first = if pos(&floats, neg_zero.to_bits()) < pos(&floats, 0) {
        "-0"
    } else {
        "0"
    };
    println!(
        "          (to Go's rule -0 == 0, so they keep their input order, and {first} came first)"
    );
    let not_nan = |xs: &[f64]| -> Vec<f64> { xs.iter().copied().filter(|x| !x.is_nan()).collect() };
    let (r, t) = (not_nan(&rad), not_nan(&go));
    let differ: Vec<String> = (0..r.len())
        .filter(|&i| r[i].to_bits() != t[i].to_bits())
        .map(|i| format!("{i} ({} vs {})", show(r[i]), show(t[i])))
        .collect();
    println!(
        "    drop the NaNs from both: {} of {} positions match; the rest: {}",
        r.len() - differ.len(),
        r.len(),
        differ.join(", ")
    );
}
```

**Rust output:**

```
1) i64
  input:              [5 -1 -9223372036854775808 0 9223372036854775807 -42 42 1 -2]
  radix on raw bits:  [0 1 5 42 9223372036854775807 -9223372036854775808 -42 -2 -1]  <- negatives after positives

                     x          raw bits         x ^ 1<<63
                     5  0000000000000005  8000000000000005
                    -1  ffffffffffffffff  7fffffffffffffff
  -9223372036854775808  8000000000000000  0000000000000000
                     0  0000000000000000  8000000000000000
   9223372036854775807  7fffffffffffffff  ffffffffffffffff
                   -42  ffffffffffffffd6  7fffffffffffffd6
                    42  000000000000002a  800000000000002a
                     1  0000000000000001  8000000000000001
                    -2  fffffffffffffffe  7ffffffffffffffe

  radix on x ^ 1<<63: [-9223372036854775808 -42 -2 -1 0 1 5 42 9223372036854775807]
  matches sort_unstable: true

2) f64
  the literal -0.0 has its sign bit set: true; 0.0_f64.copysign(-1.0) does: true

         x          raw bits          key bits
      3.14  40091eb851eb851f  c0091eb851eb851f
      -2.5  c004000000000000  3ffbffffffffffff
         0  0000000000000000  8000000000000000
        -0  8000000000000000  7fffffffffffffff
      +Inf  7ff0000000000000  fff0000000000000
      -Inf  fff0000000000000  000fffffffffffff
    5e-324  0000000000000001  8000000000000001
   -5e-324  8000000000000001  7ffffffffffffffe
       NaN  7ff8000000000000  fff8000000000000
      -NaN  fff8000000000000  0007ffffffffffff
         1  3ff0000000000000  bff0000000000000
        -1  bff0000000000000  400fffffffffffff
  key -> float -> key round-trips every value bit for bit: true

  radix (totalOrder):           [-NaN -Inf -2.5 -1 -5e-324 -0 0 5e-324 1 3.14 +Inf NaN]
  Go's cmp.Less, stable sort:   [NaN -NaN -Inf -2.5 -1 -5e-324 0 -0 5e-324 1 3.14 +Inf]
  Go's rule agrees with radix, bit for bit: false
  sort_unstable_by(total_cmp):  [-NaN -Inf -2.5 -1 -5e-324 -0 0 5e-324 1 3.14 +Inf NaN]
  total_cmp agrees with radix, bit for bit: true
  total_cmp(a, b) == key(a).cmp(key(b)) for all 144 pairs: true

  where radix and Go's rule disagree:
    NaN:  radix puts -NaN at 0 and NaN at 11 (by sign); Go's rule puts them at 1 and 0
    zero: radix puts -0 at 5 and 0 at 6; Go's rule puts them at 7 and 6
          (to Go's rule -0 == 0, so they keep their input order, and 0 came first)
    drop the NaNs from both: 8 of 10 positions match; the rest: 4 (-0 vs 0), 5 (0 vs -0)
```

> **Go vs Rust:** Every bit pattern, the radix order and the disagreement report are identical to Go's (`f64::to_bits` = `math.Float64bits`). The stdlib line is where the languages part: Rust's `sort_unstable_by(f64::total_cmp)` **agrees** with the radix order, and the program checks why — `total_cmp` equals comparing the transformed keys on all 144 pairs, because its source is this transform; Go's `slices.Sort` disagrees on both NaNs and on ±0, and the Rust twin reproduces Go's order from `cmp.Less` under a stable `sort_by`. `v.sort()` on `f64` does not compile (`f64: !Ord`), and Rust's literal `-0.0` is a real negative zero where Go's constant `-0.0` is +0.

**Complexity:** transform Θ(1) per key, so `int64`/`float64` cost the same Θ(d·(n + 2ᵇ)) as `uint64` · the radix order is IEEE totalOrder (−NaN first, +NaN last, −0 before +0); `slices.Sort` agrees on every other value but puts both NaNs first and keeps ±0 in input order. Comparing the transformed bits with `SortFunc` reproduces the radix order exactly — Go's stdlib has no total-order float comparator of its own.

---

## 9. Strings: LSD for fixed width, MSD for the rest

`🟡 medium` · *String radix sorts*

A fixed-width string is a W-digit number in base 256, so LSD sorts it in W stable passes, right to
left: Θ(W·(n + R)). Variable-length strings need **MSD**: counting-sort on byte d, then recurse into
each bucket on byte d + 1, with "end of string" as a bucket *below* every byte so that `she` comes
before `shells`. MSD reads only each string's distinguishing prefix — Θ(D) byte reads — but every call
clears and scans R + 2 counters, and on thousands of tiny buckets that fixed cost dwarfs the bytes. An
insertion-sort cutoff for small buckets is the fix, and the counts show both sides of it: too small
and the counters dominate, too large and insertion sort goes quadratic.

**Steps:**

1. LSD-sort ten ISO dates (W = 10) and print the array when the day, then month-day, then the whole date is sorted.
2. MSD-sort Sedgewick's "she sells seashells" sentence with the end-of-string bucket, counting calls, counter slots, bytes inspected and insertion compares, with and without a cutoff.
3. Repeat on 10,000 generated words for cutoffs from 0 to 512, and find the cutoff with the least total work.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
)

const R = 256 // radix: one bucket per byte value

// ---------- LSD: fixed-width strings are W-digit numbers in base 256 ----------

// lsdStrings sorts strings that all have length w, right to left, one stable
// counting sort per byte position. snap is called after every pass.
func lsdStrings(a []string, w int, inspected *int, snap func(d int, a []string)) {
	aux := make([]string, len(a))
	for d := w - 1; d >= 0; d-- {
		var count [R + 1]int // count[c+1] counts byte c
		for _, s := range a {
			count[int(s[d])+1]++
			*inspected++
		}
		for r := range R { // prefix sum: count[c] = start of byte c's bucket
			count[r+1] += count[r]
		}
		for _, s := range a { // stable placement (reads s[d] a second time)
			aux[count[s[d]]] = s
			count[s[d]]++
		}
		copy(a, aux)
		snap(d, a)
	}
}

// ---------- MSD: variable length, recurse into each bucket ----------

type stats struct {
	calls     int // radix calls (each pays for R+2 counters)
	slots     int // counter slots those calls allocate and clear
	inspected int // byte positions read by the radix calls (each read twice: count, place)
	ends      int // ...of which past the end of the string (no byte there)
	compares  int // byte positions compared inside insertion sort
}

func (s stats) total() int { return s.slots + 2*s.inspected + s.compares }

// charAt is Sedgewick's convention: -1 past the end, so "end of string" is a
// bucket BELOW every byte and "she" sorts before "shells".
func charAt(s string, d int) int {
	if d < len(s) {
		return int(s[d])
	}
	return -1
}

// less compares two strings known to share their first d bytes.
func less(a, b string, d int, st *stats) bool {
	for i := d; i < len(a) && i < len(b); i++ {
		st.compares++
		if a[i] != b[i] {
			return a[i] < b[i]
		}
	}
	return len(a) < len(b)
}

func insertion(a []string, lo, hi, d int, st *stats) {
	for i := lo + 1; i < hi; i++ {
		for j := i; j > lo && less(a[j], a[j-1], d, st); j-- {
			a[j], a[j-1] = a[j-1], a[j]
		}
	}
}

// msd sorts a[lo:hi], whose strings all share their first d bytes.
func msd(a, aux []string, lo, hi, d, cutoff int, st *stats) {
	if hi-lo <= cutoff {
		insertion(a, lo, hi, d, st)
		return
	}
	st.calls++
	var count [R + 2]int // bucket b = charAt+1 (0 = ended) is counted in count[b+1]
	st.slots += len(count)
	for i := lo; i < hi; i++ {
		c := charAt(a[i], d)
		count[c+2]++
		st.inspected++
		if c < 0 {
			st.ends++
		}
	}
	for b := range R + 1 { // count[b] = start of bucket b
		count[b+1] += count[b]
	}
	for i := lo; i < hi; i++ { // now count[b] ends up at the END of bucket b
		b := charAt(a[i], d) + 1
		aux[count[b]] = a[i]
		count[b]++
	}
	copy(a[lo:hi], aux[:hi-lo])
	// Bucket 0 holds strings that ended at d: all equal, nothing to do.
	// Every other bucket shares d+1 bytes now; recurse if it has 2 or more.
	for b := 1; b <= R; b++ {
		if s, e := lo+count[b-1], lo+count[b]; e-s >= 2 {
			msd(a, aux, s, e, d+1, cutoff, st)
		}
	}
}

func msdSort(words []string, cutoff int) ([]string, stats) {
	a := slices.Clone(words)
	var st stats
	msd(a, make([]string, len(a)), 0, len(a), 0, cutoff, &st)
	return a, st
}

func statsTable(words []string, cutoffs []int) {
	total := 0
	for _, w := range words {
		total += len(w)
	}
	ref := slices.Clone(words)
	slices.Sort(ref)
	fmt.Printf("  %6s  %7s  %13s  %15s  %16s  %10s\n",
		"cutoff", "calls", "counter slots", "bytes inspected", "insertion cmps", "total")
	ok := true
	read := 0
	bestCut, bestTotal := 0, 0
	for _, c := range cutoffs {
		out, st := msdSort(words, c)
		ok = ok && slices.Equal(out, ref)
		if c == 0 {
			read = st.inspected - st.ends
		}
		fmt.Printf("  %6d  %7d  %13d  %15d  %16d  %10d\n",
			c, st.calls, st.slots, st.inspected, st.compares, st.total())
		if bestTotal == 0 || st.total() < bestTotal {
			bestCut, bestTotal = c, st.total()
		}
	}
	fmt.Println("  (total = counter slots + 2 * bytes inspected + insertion compares)")
	fmt.Printf("  smallest total at cutoff %d; every row equals slices.Sort: %v\n", bestCut, ok)
	fmt.Printf("  cutoff 0 read %d of the %d input bytes and never touched the other %d (%.0f%%)\n",
		read, total, total-read, 100*float64(total-read)/float64(total))
}

func main() {
	fmt.Println("1) LSD on fixed-width keys: ISO dates, W = 10 bytes, right to left")
	dates := []string{"2026-11-09", "2025-03-14", "2026-01-31", "1999-12-31", "2026-11-02",
		"2025-03-01", "2000-01-01", "2026-01-09", "1999-01-31", "2025-11-09"}
	cols := [][]string{slices.Clone(dates)}
	inspected := 0
	a := slices.Clone(dates)
	lsdStrings(a, 10, &inspected, func(d int, a []string) {
		if d == 8 || d == 5 || d == 0 { // a field just became fully sorted
			cols = append(cols, slices.Clone(a))
		}
	})
	fmt.Println()
	fmt.Printf("  %-10s   %-10s   %-10s   %-10s\n", "input", "by DD", "by MM-DD", "by all")
	fmt.Printf("  %-10s   %-10s   %-10s   %-10s\n", "", "(d = 8)", "(d = 5)", "(d = 0)")
	for i := range dates {
		fmt.Printf("  %s   %s   %s   %s\n", cols[0][i], cols[1][i], cols[2][i], cols[3][i])
	}
	ref := slices.Clone(dates)
	slices.Sort(ref)
	fmt.Println()
	fmt.Printf("  passes: 10, byte positions inspected: %d = W*n (each read twice: count, place)\n", inspected)
	fmt.Printf("  counter slots: %d = W*(R+1); equals slices.Sort: %v\n", 10*(R+1), slices.Equal(a, ref))
	var constant []string
	for d := range 10 {
		if !slices.ContainsFunc(dates, func(s string) bool { return s[d] != dates[0][d] }) {
			constant = append(constant, fmt.Sprintf("%d (%q)", d, dates[0][d]))
		}
	}
	fmt.Printf("  constant positions: %s -- those passes move every string and change nothing\n",
		strings.Join(constant, ", "))

	fmt.Println()
	fmt.Println("2) MSD on variable-length words (Sedgewick & Wayne, section 5.1)")
	words := strings.Fields("she sells seashells by the sea shore the shells she sells are surely seashells")
	sorted, _ := msdSort(words, 0)
	fmt.Println()
	fmt.Printf("  %v\n", sorted)
	fmt.Printf("  %d words\n\n", len(words))
	statsTable(words, []int{0, 8})

	fmt.Println()
	fmt.Println("3) MSD on 10,000 generated words (PCG seed 1,2; 3-10 letters from a-z)")
	rng := rand.New(rand.NewPCG(1, 2))
	big := make([]string, 10_000)
	for i := range big {
		b := make([]byte, 3+rng.Uint64()%8)
		for j := range b {
			b[j] = 'a' + byte(rng.Uint64()%26)
		}
		big[i] = string(b)
	}
	fmt.Println()
	statsTable(big, []int{0, 4, 8, 16, 32, 512})
}
```

**Output:**

```
1) LSD on fixed-width keys: ISO dates, W = 10 bytes, right to left

  input        by DD        by MM-DD     by all    
               (d = 8)      (d = 5)      (d = 0)   
  2026-11-09   2025-03-01   2000-01-01   1999-01-31
  2025-03-14   2000-01-01   2026-01-09   1999-12-31
  2026-01-31   2026-11-02   2026-01-31   2000-01-01
  1999-12-31   2026-11-09   1999-01-31   2025-03-01
  2026-11-02   2026-01-09   2025-03-01   2025-03-14
  2025-03-01   2025-11-09   2025-03-14   2025-11-09
  2000-01-01   2025-03-14   2026-11-02   2026-01-09
  2026-01-09   2026-01-31   2026-11-09   2026-01-31
  1999-01-31   1999-12-31   2025-11-09   2026-11-02
  2025-11-09   1999-01-31   1999-12-31   2026-11-09

  passes: 10, byte positions inspected: 100 = W*n (each read twice: count, place)
  counter slots: 2570 = W*(R+1); equals slices.Sort: true
  constant positions: 4 ('-'), 7 ('-') -- those passes move every string and change nothing

2) MSD on variable-length words (Sedgewick & Wayne, section 5.1)

  [are by sea seashells seashells sells sells she she shells shore surely the the]
  14 words

  cutoff    calls  counter slots  bytes inspected    insertion cmps       total
       0       18           4644               63                 0        4770
       8        2            516               24                23         587
  (total = counter slots + 2 * bytes inspected + insertion compares)
  smallest total at cutoff 8; every row equals slices.Sort: true
  cutoff 0 read 54 of the 65 input bytes and never touched the other 11 (17%)

3) MSD on 10,000 generated words (PCG seed 1,2; 3-10 letters from a-z)

  cutoff    calls  counter slots  bytes inspected    insertion cmps       total
       0     2727         703566            34439                 0      772444
       4      711         183438            30040              2014      245532
       8      677         174666            29811              2451      236739
      16      238          61404            24027             24849      134307
      32       27           6966            20000             46298       93264
     512        1            258            10000           1023998     1044256
  (total = counter slots + 2 * bytes inspected + insertion compares)
  smallest total at cutoff 32; every row equals slices.Sort: true
  cutoff 0 read 33873 of the 64840 input bytes and never touched the other 30967 (48%)
```

**Rust:**

```rust
const R: usize = 256; // radix: one bucket per byte value

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state), so that
// seed (1, 2) generates exactly the words the Go program sorts.
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

// ---------- LSD: fixed-width strings are W-digit numbers in base 256 ----------

/// Sorts strings that all have length w, right to left, one stable counting
/// sort per byte position. `snap` is called after every pass.
/// The snapshot gets `&[&'a str]`, views with the strings' own lifetime, so
/// the caller may keep copies of them after the pass (it collects columns).
fn lsd_strings<'a>(
    a: &mut [&'a str],
    w: usize,
    inspected: &mut usize,
    mut snap: impl FnMut(usize, &[&'a str]),
) {
    let mut aux = vec![""; a.len()];
    for d in (0..w).rev() {
        let mut count = [0usize; R + 1]; // count[c+1] counts byte c
        for s in a.iter() {
            count[usize::from(s.as_bytes()[d]) + 1] += 1;
            *inspected += 1;
        }
        for r in 0..R {
            // prefix sum: count[c] = start of byte c's bucket
            count[r + 1] += count[r];
        }
        for &s in a.iter() {
            // stable placement (reads s[d] a second time)
            let c = usize::from(s.as_bytes()[d]);
            aux[count[c]] = s;
            count[c] += 1;
        }
        a.copy_from_slice(&aux); // &str is Copy: this copies (ptr, len) pairs
        snap(d, a);
    }
}

// ---------- MSD: variable length, recurse into each bucket ----------

#[derive(Default)]
struct Stats {
    calls: usize,     // radix calls (each pays for R+2 counters)
    slots: usize,     // counter slots those calls allocate and clear
    inspected: usize, // byte positions read by the radix calls (each read twice: count, place)
    ends: usize,      // ...of which past the end of the string (no byte there)
    compares: usize,  // byte positions compared inside insertion sort
}

impl Stats {
    fn total(&self) -> usize {
        self.slots + 2 * self.inspected + self.compares
    }
}

/// The byte at d as `Option` -- `None` past the end. Go returns a -1 sentinel;
/// here the type says "there may be no byte".
fn char_at(s: &str, d: usize) -> Option<u8> {
    s.as_bytes().get(d).copied()
}

/// Bucket 0 = "ended", below every byte, so "she" sorts before "shells".
fn bucket(c: Option<u8>) -> usize {
    c.map_or(0, |b| usize::from(b) + 1)
}

/// Compares two strings known to share their first d bytes.
fn less(a: &str, b: &str, d: usize, st: &mut Stats) -> bool {
    let (a, b) = (a.as_bytes(), b.as_bytes());
    for i in d..a.len().min(b.len()) {
        st.compares += 1;
        if a[i] != b[i] {
            return a[i] < b[i];
        }
    }
    a.len() < b.len()
}

fn insertion(a: &mut [&str], d: usize, st: &mut Stats) {
    for i in 1..a.len() {
        let mut j = i;
        while j > 0 && less(a[j], a[j - 1], d, st) {
            a.swap(j, j - 1);
            j -= 1;
        }
    }
}

/// Sorts `a`, whose strings all share their first d bytes. Go passes the
/// whole slice plus lo/hi; here each call gets its own subslice, and `aux` is
/// re-sliced to match, so there is no index arithmetic to get wrong.
/// The `'a` says that `a` and `aux` hold views into the same strings: views
/// move both ways between them, so the compiler must know they live as long.
fn msd<'a>(a: &mut [&'a str], aux: &mut [&'a str], d: usize, cutoff: usize, st: &mut Stats) {
    let n = a.len();
    if n <= cutoff {
        insertion(a, d, st);
        return;
    }
    st.calls += 1;
    let mut count = [0usize; R + 2]; // bucket b is counted in count[b+1]
    st.slots += count.len();
    for s in a.iter() {
        let c = char_at(s, d);
        count[bucket(c) + 1] += 1;
        st.inspected += 1;
        if c.is_none() {
            st.ends += 1;
        }
    }
    for b in 0..=R {
        // count[b] = start of bucket b
        count[b + 1] += count[b];
    }
    for &s in a.iter() {
        // now count[b] ends up at the END of bucket b
        let b = bucket(char_at(s, d));
        aux[count[b]] = s;
        count[b] += 1;
    }
    a.copy_from_slice(&aux[..n]);
    // Bucket 0 holds strings that ended at d: all equal, nothing to do.
    // Every other bucket shares d+1 bytes now; recurse if it has 2 or more.
    for b in 1..=R {
        let (s, e) = (count[b - 1], count[b]);
        if e - s >= 2 {
            msd(&mut a[s..e], &mut aux[..e - s], d + 1, cutoff, st);
        }
    }
}

fn msd_sort<'a>(words: &[&'a str], cutoff: usize) -> (Vec<&'a str>, Stats) {
    let mut a = words.to_vec();
    let mut aux = vec![""; a.len()];
    let mut st = Stats::default();
    msd(&mut a, &mut aux, 0, cutoff, &mut st);
    (a, st)
}

fn stats_table(words: &[&str], cutoffs: &[usize]) {
    let total: usize = words.iter().map(|w| w.len()).sum();
    let mut reference = words.to_vec();
    reference.sort_unstable(); // str's Ord is byte-wise, like Go's
    println!(
        "  {:>6}  {:>7}  {:>13}  {:>15}  {:>16}  {:>10}",
        "cutoff", "calls", "counter slots", "bytes inspected", "insertion cmps", "total"
    );
    let mut ok = true;
    let mut read = 0;
    let mut best: Option<(usize, usize)> = None; // (cutoff, total); first wins ties, like Go
    for &c in cutoffs {
        let (out, st) = msd_sort(words, c);
        ok &= out == reference;
        if c == 0 {
            read = st.inspected - st.ends;
        }
        println!(
            "  {:>6}  {:>7}  {:>13}  {:>15}  {:>16}  {:>10}",
            c,
            st.calls,
            st.slots,
            st.inspected,
            st.compares,
            st.total()
        );
        if best.is_none_or(|(_, t)| st.total() < t) {
            best = Some((c, st.total()));
        }
    }
    println!("  (total = counter slots + 2 * bytes inspected + insertion compares)");
    println!(
        "  smallest total at cutoff {}; every row equals sort_unstable: {ok}",
        best.unwrap().0
    );
    println!(
        "  cutoff 0 read {read} of the {total} input bytes and never touched the other {} ({:.0}%)",
        total - read,
        100.0 * (total - read) as f64 / total as f64
    );
}

fn go_slice(xs: &[&str]) -> String {
    format!("[{}]", xs.join(" "))
}

fn main() {
    println!("1) LSD on fixed-width keys: ISO dates, W = 10 bytes, right to left");
    let dates = [
        "2026-11-09",
        "2025-03-14",
        "2026-01-31",
        "1999-12-31",
        "2026-11-02",
        "2025-03-01",
        "2000-01-01",
        "2026-01-09",
        "1999-01-31",
        "2025-11-09",
    ];
    let mut cols = vec![dates.to_vec()];
    let mut inspected = 0;
    let mut a = dates.to_vec();
    lsd_strings(&mut a, 10, &mut inspected, |d, a| {
        if d == 8 || d == 5 || d == 0 {
            // a field just became fully sorted
            cols.push(a.to_vec());
        }
    });
    println!();
    println!(
        "  {:<10}   {:<10}   {:<10}   {:<10}",
        "input", "by DD", "by MM-DD", "by all"
    );
    println!(
        "  {:<10}   {:<10}   {:<10}   {:<10}",
        "", "(d = 8)", "(d = 5)", "(d = 0)"
    );
    let [c0, c1, c2, c3] = &cols[..] else {
        unreachable!("input + three snapshots")
    };
    for (((w, x), y), z) in c0.iter().zip(c1).zip(c2).zip(c3) {
        println!("  {w}   {x}   {y}   {z}");
    }
    let mut reference = dates.to_vec();
    reference.sort_unstable();
    println!();
    println!(
        "  passes: 10, byte positions inspected: {inspected} = W*n (each read twice: count, place)"
    );
    println!(
        "  counter slots: {} = W*(R+1); equals sort_unstable: {}",
        10 * (R + 1),
        a == reference
    );
    let first = dates[0].as_bytes();
    let constant: Vec<String> = (0..10)
        .filter(|&d| dates.iter().all(|s| s.as_bytes()[d] == first[d]))
        .map(|d| format!("{d} ('{}')", first[d] as char))
        .collect();
    println!(
        "  constant positions: {} -- those passes move every string and change nothing",
        constant.join(", ")
    );

    println!();
    println!("2) MSD on variable-length words (Sedgewick & Wayne, section 5.1)");
    let words: Vec<&str> =
        "she sells seashells by the sea shore the shells she sells are surely seashells"
            .split_whitespace()
            .collect();
    let (sorted, _) = msd_sort(&words, 0);
    println!();
    println!("  {}", go_slice(&sorted));
    println!("  {} words\n", words.len());
    stats_table(&words, &[0, 8]);

    println!();
    println!("3) MSD on 10,000 generated words (PCG seed 1,2; 3-10 letters from a-z)");
    let mut rng = Pcg::new(1, 2);
    let owned: Vec<String> = (0..10_000)
        .map(|_| {
            let len = 3 + rng.next_u64() % 8;
            (0..len)
                .map(|_| char::from(b'a' + (rng.next_u64() % 26) as u8))
                .collect()
        })
        .collect();
    // The sort moves &str views; the Strings stay put and own the bytes.
    let big: Vec<&str> = owned.iter().map(String::as_str).collect();
    println!();
    stats_table(&big, &[0, 4, 8, 16, 32, 512]);
}
```

**Rust output:**

```
1) LSD on fixed-width keys: ISO dates, W = 10 bytes, right to left

  input        by DD        by MM-DD     by all    
               (d = 8)      (d = 5)      (d = 0)   
  2026-11-09   2025-03-01   2000-01-01   1999-01-31
  2025-03-14   2000-01-01   2026-01-09   1999-12-31
  2026-01-31   2026-11-02   2026-01-31   2000-01-01
  1999-12-31   2026-11-09   1999-01-31   2025-03-01
  2026-11-02   2026-01-09   2025-03-01   2025-03-14
  2025-03-01   2025-11-09   2025-03-14   2025-11-09
  2000-01-01   2025-03-14   2026-11-02   2026-01-09
  2026-01-09   2026-01-31   2026-11-09   2026-01-31
  1999-01-31   1999-12-31   2025-11-09   2026-11-02
  2025-11-09   1999-01-31   1999-12-31   2026-11-09

  passes: 10, byte positions inspected: 100 = W*n (each read twice: count, place)
  counter slots: 2570 = W*(R+1); equals sort_unstable: true
  constant positions: 4 ('-'), 7 ('-') -- those passes move every string and change nothing

2) MSD on variable-length words (Sedgewick & Wayne, section 5.1)

  [are by sea seashells seashells sells sells she she shells shore surely the the]
  14 words

  cutoff    calls  counter slots  bytes inspected    insertion cmps       total
       0       18           4644               63                 0        4770
       8        2            516               24                23         587
  (total = counter slots + 2 * bytes inspected + insertion compares)
  smallest total at cutoff 8; every row equals sort_unstable: true
  cutoff 0 read 54 of the 65 input bytes and never touched the other 11 (17%)

3) MSD on 10,000 generated words (PCG seed 1,2; 3-10 letters from a-z)

  cutoff    calls  counter slots  bytes inspected    insertion cmps       total
       0     2727         703566            34439                 0      772444
       4      711         183438            30040              2014      245532
       8      677         174666            29811              2451      236739
      16      238          61404            24027             24849      134307
      32       27           6966            20000             46298       93264
     512        1            258            10000           1023998     1044256
  (total = counter slots + 2 * bytes inspected + insertion compares)
  smallest total at cutoff 32; every row equals sort_unstable: true
  cutoff 0 read 33873 of the 64840 input bytes and never touched the other 30967 (48%)
```

> **Go vs Rust:** Output identical to Go's apart from the oracle's name: same words (PCG port), same calls, slots, bytes and compares. The sort moves `&str` views — a pointer and a length, `Copy`, like a Go string header — over `String`s that never move; the borrow checker then wants a named lifetime `'a` on `msd` because views flow both ways between `a` and `aux`. Go's `-1` past-the-end sentinel becomes `s.as_bytes().get(d)` → `Option<u8>`, and each recursive call takes its bucket as a subslice `&mut a[s..e]` instead of `lo`/`hi` indices. The oracle `sort_unstable` agrees because `str`'s `Ord` is byte-wise, as Go's string comparison is.

**Complexity:** LSD Θ(W·(n + R)) — 10 passes, W·n = 100 byte positions for the dates · MSD Θ(D) byte reads + Θ(R) per call, space Θ(n + depth·R). On 10,000 random words MSD without a cutoff made 2,727 calls and cleared 703,566 counter slots for 64,840 bytes of input — and never read 48% of those bytes; a cutoff of 32 cut the counters 100-fold (6,966) and the total work about 8-fold, while a cutoff of 512 sent buckets of hundreds to insertion sort and cost over a million compares.

---

## 10. Bucket sort and the uniform assumption

`🟡 medium` · *Bucket sort / distribution*

Bucket sort spreads n keys from [0, 1) over n equal-width buckets, insertion-sorts each bucket and
concatenates. Its cost is the sum of the insertion sorts, about Σ nᵢ²/4 comparisons, and for **uniform**
keys every nᵢ is Poisson(1)-distributed, so E[Σ nᵢ²] = 2n − 1 and the whole sort is expected **Θ(n)**.
Nothing in the code checks that assumption: feed it a skewed shape (x⁸) or a narrow range and the same
code piles thousands of keys into a few buckets and goes **Θ(n²)**. Normalising by [min, max] rescues a
narrow range, because its shape was uniform all along; it cannot rescue a skewed shape. The counts below
make each of those claims a number.

**Steps:**

1. Draw 100,000 uniform keys from a seeded PCG and print the bucket-occupancy histogram next to Poisson(1).
2. Reshape the same keys (x², x⁸, a narrow range) and count Σ nᵢ² and the in-bucket comparisons for each.
3. Normalise the narrow range and x⁸ by [min, max] and see which one comes back to the uniform numbers.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
)

// Bucket sort for keys in [0, 1): n equal-width buckets, key x goes to bucket
// floor(x*n), insertion-sort every bucket, concatenate.
//
// The cost is the sum of the insertion sorts, about sum(n_i^2)/4 comparisons.
// For UNIFORM keys each bucket count n_i is Binomial(n, 1/n) ~ Poisson(1), and
// E[n_i^2] = Var + mean^2 = (1 - 1/n) + 1, so E[sum n_i^2] = 2n - 1: linear.
// Nothing in the code checks that the keys are uniform. That assumption IS
// the algorithm -- break it and the very same code is quadratic.

type stats struct {
	maxBucket int // largest bucket
	sumSq     int // sum of n_i^2
	cmps      int // key comparisons inside the insertion sorts
	nonEmpty  int // buckets holding at least one key
	occ       [6]int
}

// insertionSort sorts a and returns the number of key comparisons it made.
func insertionSort(a []float64) int {
	cmps := 0
	for i := 1; i < len(a); i++ {
		key, j := a[i], i
		for j > 0 {
			cmps++
			if a[j-1] <= key {
				break
			}
			a[j] = a[j-1] // shift right
			j--
		}
		a[j] = key
	}
	return cmps
}

// bucketSort sorts xs (all in [0, 1)) into a new slice.
func bucketSort(xs []float64) ([]float64, stats) {
	n := len(xs)
	buckets := make([][]float64, n)
	for _, x := range xs {
		i := int(x * float64(n)) // x < 1, so i <= n-1
		buckets[i] = append(buckets[i], x)
	}
	var st stats
	out := make([]float64, 0, n)
	for _, b := range buckets {
		st.cmps += insertionSort(b)
		st.sumSq += len(b) * len(b)
		st.maxBucket = max(st.maxBucket, len(b))
		st.occ[min(len(b), 5)]++
		if len(b) > 0 {
			st.nonEmpty++
		}
		out = append(out, b...)
	}
	return out, st
}

// normalise maps [min, max] onto [0, 1); max itself lands in the last bucket.
func normalise(xs []float64) []float64 {
	lo, hi := slices.Min(xs), slices.Max(xs)
	out := make([]float64, len(xs))
	for i, x := range xs {
		out[i] = min((x-lo)/(hi-lo), math.Nextafter(1, 0))
	}
	return out
}

func main() {
	const n = 100_000
	r := rand.New(rand.NewPCG(1, 2))
	uni := make([]float64, n)
	for i := range uni {
		uni[i] = r.Float64()
	}

	runs, allOK := 0, true
	run := func(xs []float64) stats {
		got, st := bucketSort(xs)
		want := slices.Clone(xs)
		slices.Sort(want)
		runs++
		allOK = allOK && slices.Equal(got, want)
		return st
	}

	// 1. Uniform keys: the occupancy histogram is the Poisson(1) picture.
	st := run(uni)
	fmt.Printf("1. uniform keys, n = %d, %d buckets\n\n", n, n)
	fmt.Printf("   %-12s %8s %7s   %s\n", "keys/bucket", "buckets", "%", "Poisson(1): n*e^-1/k!")
	fact, cum := 1.0, 0.0
	for k := 0; k <= 5; k++ {
		if k > 0 {
			fact *= float64(k)
		}
		label, exp := fmt.Sprint(k), n*math.Exp(-1)/fact
		if k == 5 {
			label, exp = "5+", n-cum
		}
		cum += exp
		fmt.Printf("   %-12s %8d %6.2f%%   %10.1f\n", label, st.occ[k], 100*float64(st.occ[k])/n, exp)
	}
	fmt.Printf("\n   sum n_i^2 = %d   (expected 2n - 1 = %d)\n", st.sumSq, 2*n-1)
	fmt.Printf("   in-bucket comparisons = %d (%.2f per key), max bucket = %d keys\n",
		st.cmps, float64(st.cmps)/n, st.maxBucket)
	uniCmps := st.cmps

	// 2. Same keys, different shapes. Powers by multiplication (no math.Pow:
	// libm results may differ between languages in the last bit).
	sq, p8, narrow := make([]float64, n), make([]float64, n), make([]float64, n)
	for i, x := range uni {
		sq[i] = x * x
		x4 := sq[i] * sq[i]
		p8[i] = x4 * x4
		narrow[i] = 0.5 + x/10000
	}
	rows := []struct {
		name string
		xs   []float64
	}{
		{"uniform x", uni},
		{"x^2", sq},
		{"x^8", p8},
		{"0.5 + x/10^4", narrow},
		{"0.5 + x/10^4, normalised", normalise(narrow)},
		{"x^8, normalised", normalise(p8)},
	}
	fmt.Printf("\n2-3. same %d keys, reshaped (rows 5-6: normalised by [min, max] first)\n\n", n)
	fmt.Printf("   %-26s %10s %12s %14s %10s\n", "distribution", "max bucket", "sum n_i^2", "comparisons", "cmps / n")
	res := make([]stats, len(rows))
	for i, row := range rows {
		st := run(row.xs)
		res[i] = st
		fmt.Printf("   %-26s %10d %12d %14d %10.1f\n", row.name, st.maxBucket, st.sumSq, st.cmps, float64(st.cmps)/n)
	}

	ratio := func(i int) float64 { return float64(res[i].cmps) / float64(uniCmps) }
	fmt.Println()
	fmt.Printf("   x^8 costs %.0fx the uniform comparisons: P(x^8 < 1/n) = n^(-1/8), and\n", ratio(2))
	fmt.Printf("   bucket 0 alone holds %d keys (%.1f%%), insertion-sorted in Theta(m^2).\n",
		res[2].maxBucket, 100*float64(res[2].maxBucket)/n)
	fmt.Printf("   the narrow range costs %.0fx: all keys land in %d of the %d buckets.\n",
		ratio(3), res[3].nonEmpty, n)
	fmt.Printf("   normalised, the narrow range is back to %.2fx uniform -- its SHAPE was uniform;\n", ratio(4))
	fmt.Printf("   normalised, x^8 is still %.0fx -- [min, max] fixes a range, not a shape.\n", ratio(5))

	// 4. Every run above was checked against slices.Sort.
	fmt.Printf("\n4. all %d bucket-sort outputs equal slices.Sort: %v\n", runs, allOK)
}
```

**Output:**

```
1. uniform keys, n = 100000, 100000 buckets

   keys/bucket   buckets       %   Poisson(1): n*e^-1/k!
   0               36729  36.73%      36787.9
   1               36886  36.89%      36787.9
   2               18386  18.39%      18394.0
   3                6085   6.08%       6131.3
   4                1551   1.55%       1532.8
   5+                363   0.36%        366.0

   sum n_i^2 = 199846   (expected 2n - 1 = 199999)
   in-bucket comparisons = 45235 (0.45 per key), max bucket = 7 keys

2-3. same 100000 keys, reshaped (rows 5-6: normalised by [min, max] first)

   distribution               max bucket    sum n_i^2    comparisons   cmps / n
   uniform x                           7       199846          45235        0.5
   x^2                               309       476912         122191        1.2
   x^8                             23560    566755300      141996346     1420.0
   0.5 + x/10^4                    10133   1000055096      249465216     2494.7
   0.5 + x/10^4, normalised            7       200130          45319        0.5
   x^8, normalised                 23560    566752902      141994930     1419.9

   x^8 costs 3139x the uniform comparisons: P(x^8 < 1/n) = n^(-1/8), and
   bucket 0 alone holds 23560 keys (23.6%), insertion-sorted in Theta(m^2).
   the narrow range costs 5515x: all keys land in 10 of the 100000 buckets.
   normalised, the narrow range is back to 1.00x uniform -- its SHAPE was uniform;
   normalised, x^8 is still 3139x -- [min, max] fixes a range, not a shape.

4. all 7 bucket-sort outputs equal slices.Sort: true
```

**Rust:**

```rust
// Bucket sort for keys in [0, 1): n equal-width buckets, key x goes to bucket
// floor(x*n), insertion-sort every bucket, concatenate.
//
// The cost is the sum of the insertion sorts, about sum(n_i^2)/4 comparisons.
// For UNIFORM keys each bucket count n_i is Binomial(n, 1/n) ~ Poisson(1), and
// E[n_i^2] = Var + mean^2 = (1 - 1/n) + 1, so E[sum n_i^2] = 2n - 1: linear.
// Nothing in the code checks that the keys are uniform. That assumption IS
// the algorithm -- break it and the very same code is quadratic.

/// Port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state): with the same
/// seed (1, 2) it produces Go's exact stream, so every bucket count matches.
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
        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }

    /// Go's Float64: the low 53 bits, divided by 2^53 -- exactly representable.
    fn float64(&mut self) -> f64 {
        ((self.next_u64() << 11) >> 11) as f64 / (1u64 << 53) as f64
    }
}

#[derive(Default)]
struct Stats {
    max_bucket: usize, // largest bucket
    sum_sq: usize,     // sum of n_i^2
    cmps: usize,       // key comparisons inside the insertion sorts
    non_empty: usize,  // buckets holding at least one key
    occ: [usize; 6],   // buckets holding 0, 1, 2, 3, 4, 5+ keys
}

/// Sorts a and returns the number of key comparisons it made.
fn insertion_sort(a: &mut [f64]) -> usize {
    let mut cmps = 0;
    for i in 1..a.len() {
        let (key, mut j) = (a[i], i);
        while j > 0 {
            cmps += 1;
            if a[j - 1] <= key {
                break;
            }
            a[j] = a[j - 1]; // shift right
            j -= 1;
        }
        a[j] = key;
    }
    cmps
}

/// Sorts xs (all in [0, 1)) into a new Vec.
fn bucket_sort(xs: &[f64]) -> (Vec<f64>, Stats) {
    let n = xs.len();
    // Vec<Vec<f64>>: n empty inner Vecs cost no allocation until first push,
    // just like Go's nil slices.
    let mut buckets: Vec<Vec<f64>> = vec![Vec::new(); n];
    for &x in xs {
        let i = (x * n as f64) as usize; // x < 1, so i <= n-1
        buckets[i].push(x);
    }
    let mut st = Stats::default();
    let mut out = Vec::with_capacity(n);
    for b in &mut buckets {
        st.cmps += insertion_sort(b);
        st.sum_sq += b.len() * b.len();
        st.max_bucket = st.max_bucket.max(b.len());
        st.occ[b.len().min(5)] += 1;
        if !b.is_empty() {
            st.non_empty += 1;
        }
        out.extend_from_slice(b);
    }
    (out, st)
}

/// Maps [min, max] onto [0, 1); max itself lands in the last bucket.
fn normalise(xs: &[f64]) -> Vec<f64> {
    // f64 is not Ord (NaN), so there is no xs.iter().min(): fold with f64::min.
    let lo = xs.iter().copied().fold(f64::INFINITY, f64::min);
    let hi = xs.iter().copied().fold(f64::NEG_INFINITY, f64::max);
    let below_one = f64::from_bits(1f64.to_bits() - 1); // Go's math.Nextafter(1, 0)
    xs.iter()
        .map(|&x| ((x - lo) / (hi - lo)).min(below_one))
        .collect()
}

fn main() {
    const N: usize = 100_000;
    let nf = N as f64;
    let mut r = Pcg::new(1, 2);
    let uni: Vec<f64> = (0..N).map(|_| r.float64()).collect();

    // The closure borrows `runs` and `all_ok` mutably until its last call;
    // after that (non-lexical lifetimes) they can be read again below.
    let (mut runs, mut all_ok) = (0, true);
    let mut run = |xs: &[f64]| -> Stats {
        let (got, st) = bucket_sort(xs);
        let mut want = xs.to_vec();
        // v.sort() does not compile for f64 (f64: !Ord); total_cmp is the total order.
        want.sort_unstable_by(f64::total_cmp);
        runs += 1;
        all_ok &= got == want;
        st
    };

    // 1. Uniform keys: the occupancy histogram is the Poisson(1) picture.
    let st = run(&uni);
    println!("1. uniform keys, n = {N}, {N} buckets\n");
    println!(
        "   {:<12} {:>8} {:>7}   Poisson(1): n*e^-1/k!",
        "keys/bucket", "buckets", "%"
    );
    let (mut fact, mut cum) = (1.0, 0.0);
    for k in 0..=5 {
        if k > 0 {
            fact *= k as f64;
        }
        let (label, exp) = if k == 5 {
            ("5+".to_string(), nf - cum)
        } else {
            (k.to_string(), nf * (-1f64).exp() / fact)
        };
        cum += exp;
        println!(
            "   {:<12} {:>8} {:>6.2}%   {:>10.1}",
            label,
            st.occ[k],
            100.0 * st.occ[k] as f64 / nf,
            exp
        );
    }
    println!(
        "\n   sum n_i^2 = {}   (expected 2n - 1 = {})",
        st.sum_sq,
        2 * N - 1
    );
    println!(
        "   in-bucket comparisons = {} ({:.2} per key), max bucket = {} keys",
        st.cmps,
        st.cmps as f64 / nf,
        st.max_bucket
    );
    let uni_cmps = st.cmps;

    // 2. Same keys, different shapes. Powers by multiplication (no powf:
    // libm results may differ between languages in the last bit).
    let sq: Vec<f64> = uni.iter().map(|&x| x * x).collect();
    let p8: Vec<f64> = sq.iter().map(|&s| (s * s) * (s * s)).collect();
    let narrow: Vec<f64> = uni.iter().map(|&x| 0.5 + x / 10000.0).collect();
    let rows: [(&str, Vec<f64>); 6] = [
        ("uniform x", uni.clone()),
        ("x^2", sq),
        ("x^8", p8.clone()),
        ("0.5 + x/10^4", narrow.clone()),
        ("0.5 + x/10^4, normalised", normalise(&narrow)),
        ("x^8, normalised", normalise(&p8)),
    ];
    println!("\n2-3. same {N} keys, reshaped (rows 5-6: normalised by [min, max] first)\n");
    println!(
        "   {:<26} {:>10} {:>12} {:>14} {:>10}",
        "distribution", "max bucket", "sum n_i^2", "comparisons", "cmps / n"
    );
    let mut res = Vec::with_capacity(rows.len());
    for (name, xs) in &rows {
        let st = run(xs);
        println!(
            "   {:<26} {:>10} {:>12} {:>14} {:>10.1}",
            name,
            st.max_bucket,
            st.sum_sq,
            st.cmps,
            st.cmps as f64 / nf
        );
        res.push(st);
    }

    let ratio = |i: usize| res[i].cmps as f64 / uni_cmps as f64;
    println!();
    println!(
        "   x^8 costs {:.0}x the uniform comparisons: P(x^8 < 1/n) = n^(-1/8), and",
        ratio(2)
    );
    println!(
        "   bucket 0 alone holds {} keys ({:.1}%), insertion-sorted in Theta(m^2).",
        res[2].max_bucket,
        100.0 * res[2].max_bucket as f64 / nf
    );
    println!(
        "   the narrow range costs {:.0}x: all keys land in {} of the {} buckets.",
        ratio(3),
        res[3].non_empty,
        N
    );
    println!(
        "   normalised, the narrow range is back to {:.2}x uniform -- its SHAPE was uniform;",
        ratio(4)
    );
    println!(
        "   normalised, x^8 is still {:.0}x -- [min, max] fixes a range, not a shape.",
        ratio(5)
    );

    // 4. Every run above was checked against sort_unstable_by(f64::total_cmp).
    println!(
        "\n4. all {runs} bucket-sort outputs equal sort_unstable_by(f64::total_cmp): {all_ok}"
    );
}
```

**Rust output:**

```
1. uniform keys, n = 100000, 100000 buckets

   keys/bucket   buckets       %   Poisson(1): n*e^-1/k!
   0               36729  36.73%      36787.9
   1               36886  36.89%      36787.9
   2               18386  18.39%      18394.0
   3                6085   6.08%       6131.3
   4                1551   1.55%       1532.8
   5+                363   0.36%        366.0

   sum n_i^2 = 199846   (expected 2n - 1 = 199999)
   in-bucket comparisons = 45235 (0.45 per key), max bucket = 7 keys

2-3. same 100000 keys, reshaped (rows 5-6: normalised by [min, max] first)

   distribution               max bucket    sum n_i^2    comparisons   cmps / n
   uniform x                           7       199846          45235        0.5
   x^2                               309       476912         122191        1.2
   x^8                             23560    566755300      141996346     1420.0
   0.5 + x/10^4                    10133   1000055096      249465216     2494.7
   0.5 + x/10^4, normalised            7       200130          45319        0.5
   x^8, normalised                 23560    566752902      141994930     1419.9

   x^8 costs 3139x the uniform comparisons: P(x^8 < 1/n) = n^(-1/8), and
   bucket 0 alone holds 23560 keys (23.6%), insertion-sorted in Theta(m^2).
   the narrow range costs 5515x: all keys land in 10 of the 100000 buckets.
   normalised, the narrow range is back to 1.00x uniform -- its SHAPE was uniform;
   normalised, x^8 is still 3139x -- [min, max] fixes a range, not a shape.

4. all 7 bucket-sort outputs equal sort_unstable_by(f64::total_cmp): true
```

> **Go vs Rust:** Every count is identical to Go's (same PCG-DXSM stream, same `Float64`, same bucket rule), so 36,729 empty buckets, Σ nᵢ² = 199,846 and 141,996,346 comparisons for x⁸ hold in both. Only the oracle differs: `f64` is not `Ord` in Rust (NaN), so `v.sort()` does not compile and the check uses `sort_unstable_by(f64::total_cmp)`; likewise `min`/`max` over the keys are folds with `f64::min`/`f64::max` instead of Go's `slices.Min`/`slices.Max`.

**Complexity:** expected Θ(n) for uniform keys (measured: Σ nᵢ² = 199,846 against 2n − 1 = 199,999, 0.45 comparisons per key) · worst case Θ(n²) with insertion-sort buckets (x⁸: 3,139× the comparisons, one bucket holds 23.6% of the keys) · space Θ(n) · stable, since insertion sort is.

---

## 11. The problems they own: maximum gap, top-K frequent, H-index

`🟡 medium` · *Classic problems*

Three interview problems where a counting or bucket sort that **stops early** is the Θ(n) answer, and
sorting (Θ(n log n)) is the brute force. **Maximum gap:** the n − 1 gaps add up to R = max − min, so the
largest is at least R/(n − 1); buckets of exactly that width can never contain it, so each bucket only
needs its min and max. **Top-K frequent:** a frequency is at most n, so frequencies are bucket indices.
**H-index:** h can never exceed n, so citations above n are capped at n and a count array of n + 1 slots
suffices. Each one is proved against a sort-based brute force on 20,000 seeded random cases, half of
them on a narrow value range where duplicates and ties are the norm.

**Steps:**

1. Work one small example of each by hand: print the buckets, the frequency buckets, the capped counts.
2. Write the sort-based brute force for each, and fix the top-K tie rule (higher frequency, then smaller value).
3. Run 20,000 random cases per problem and confirm `mismatches: 0`, then count the structures at n = 10⁶.

```go
package main

import (
	"cmp"
	"fmt"
	"maps"
	"math/rand/v2"
	"slices"
)

// Three interview problems whose Theta(n) solution IS a counting or bucket sort
// that stops early: it builds the buckets and reads the answer off them,
// without ever finishing the sort.

// ---- 1. Maximum gap: largest difference between neighbours in sorted order.
//
// Let R = max - min. The n-1 gaps of the sorted order add up to R, so the
// largest is at least R/(n-1) (pigeonhole). Cut [min, max] into buckets of
// width R/(n-1): bucket floor((x-min)*(n-1)/R), n of them (the last one holds
// only max). Two keys in the same bucket differ by LESS than R/(n-1), so the
// maximum gap is never inside a bucket -- it runs from one bucket's max to the
// next non-empty bucket's min. Each bucket only needs its min and max.

type bucket struct {
	used     bool
	min, max int
}

func maxGapBuckets(xs []int) ([]bucket, int) {
	n := len(xs)
	if n < 2 {
		return nil, 0
	}
	lo, hi := slices.Min(xs), slices.Max(xs)
	if lo == hi {
		return nil, 0
	}
	bs := make([]bucket, n)
	for _, x := range xs {
		i := (x - lo) * (n - 1) / (hi - lo) // integer floor: exact, no float
		if b := &bs[i]; b.used {
			b.min, b.max = min(b.min, x), max(b.max, x)
		} else {
			*b = bucket{true, x, x}
		}
	}
	gap, prev := 0, bs[0].max // bucket 0 always holds lo
	for _, b := range bs[1:] {
		if b.used {
			gap, prev = max(gap, b.min-prev), b.max
		}
	}
	return bs, gap
}

func maxGapSort(xs []int) int {
	s := slices.Sorted(slices.Values(xs))
	gap := 0
	for i := 1; i < len(s); i++ {
		gap = max(gap, s[i]-s[i-1])
	}
	return gap
}

// ---- 2. Top-K frequent values in [0, v): count, then bucket BY FREQUENCY.
//
// A frequency is at most n, so frequencies 1..n are bucket indices. Walking the
// values in increasing order puts smaller values first inside each bucket, so
// reading the buckets from n down gives the tie rule for free:
// higher frequency first, then smaller value.

func topK(xs []int, v, k int) ([][]int, []int) {
	freq := make([]int, v)
	for _, x := range xs {
		freq[x]++
	}
	byFreq := make([][]int, len(xs)+1)
	for val, f := range freq {
		if f > 0 {
			byFreq[f] = append(byFreq[f], val)
		}
	}
	var out []int
	for f := len(xs); f > 0 && len(out) < k; f-- {
		for _, val := range byFreq[f] {
			if len(out) < k {
				out = append(out, val)
			}
		}
	}
	return byFreq, out
}

func topKSort(xs []int, k int) []int {
	freq := map[int]int{}
	for _, x := range xs {
		freq[x]++
	}
	vals := slices.Collect(maps.Keys(freq)) // map order is random: sort next
	slices.SortFunc(vals, func(a, b int) int {
		return cmp.Or(cmp.Compare(freq[b], freq[a]), cmp.Compare(a, b))
	})
	return vals[:min(k, len(vals))]
}

// ---- 3. H-index: the largest h with at least h papers cited >= h times.
//
// h can never exceed n, so a paper with 500 citations counts the same as one
// with n: cap at n and a count array of n+1 slots suffices.

func hIndex(cites []int) ([]int, int) {
	n := len(cites)
	cnt := make([]int, n+1)
	for _, c := range cites {
		cnt[min(c, n)]++
	}
	atLeast := 0 // papers with >= h citations
	h := n
	for ; h > 0; h-- {
		atLeast += cnt[h]
		if atLeast >= h {
			break
		}
	}
	return cnt, h
}

func hIndexSort(cites []int) int {
	s := slices.Sorted(slices.Values(cites)) // ascending: walk from the top
	h := 0
	for h < len(s) && s[len(s)-1-h] >= h+1 {
		h++
	}
	return h
}

func main() {
	fmt.Println("1. maximum gap")
	gx := []int{1, 3, 4, 20, 9, 10, 2}
	bs, gap := maxGapBuckets(gx)
	n, lo, r := len(gx), slices.Min(gx), slices.Max(gx)-slices.Min(gx)
	fmt.Printf("   xs = %v, n = %d, R = %d, bucket width R/(n-1) = %.2f\n", gx, n, r, float64(r)/float64(n-1))
	for i, b := range bs {
		from := float64(lo*(n-1)+i*r) / float64(n-1) // exact integers, one division
		to := float64(lo*(n-1)+(i+1)*r) / float64(n-1)
		if b.used {
			fmt.Printf("   bucket %d [%5.2f, %5.2f)  min %2d  max %2d\n", i, from, to, b.min, b.max)
		} else {
			fmt.Printf("   bucket %d [%5.2f, %5.2f)  empty\n", i, from, to)
		}
	}
	fmt.Printf("   max gap = %d (sorted: %v)\n", gap, slices.Sorted(slices.Values(gx)))

	fmt.Println("\n2. top-K frequent (higher frequency first, then smaller value)")
	tx := []int{4, 1, 2, 2, 3, 3, 3, 4, 4, 4, 1, 5, 2, 0}
	byFreq, top := topK(tx, 6, 3)
	fmt.Printf("   xs = %v, K = 3\n", tx)
	for f := len(tx); f > 0; f-- {
		if len(byFreq[f]) > 0 {
			fmt.Printf("   frequency %d: %v\n", f, byFreq[f])
		}
	}
	fmt.Printf("   top 3 = %v  (2 and 3 tie at 3; the smaller value goes first)\n", top)

	fmt.Println("\n3. H-index")
	hx := []int{3, 0, 6, 1, 5, 25}
	cnt, h := hIndex(hx)
	fmt.Printf("   citations = %v, n = %d: cap at n, %d count slots\n", hx, len(hx), len(cnt))
	fmt.Printf("   cnt[0..n] = %v  (25 counted as %d)\n", cnt, len(hx))
	fmt.Printf("   h-index = %d\n", h)

	// Proof by brute force: 20,000 random cases per problem, half of them on a
	// narrow value range (0..7) so duplicates, equal values and ties happen.
	const cases = 20_000
	rng := rand.New(rand.NewPCG(1, 2))
	var bad [3]int
	elems := 0
	for c := range cases {
		m := int(rng.Uint64() % 201) // lengths 0..200
		span := uint64(1_000_000_000)
		if c%2 == 1 {
			span = 8
		}
		xs := make([]int, m)
		for i := range xs {
			xs[i] = int(rng.Uint64() % span)
		}
		elems += m
		if _, g := maxGapBuckets(xs); g != maxGapSort(xs) {
			bad[0]++
		}
		// top-K needs a bounded value range: reuse the keys mod 1000 (or 8).
		v := int(min(span, 1000))
		ys := make([]int, m)
		for i, x := range xs {
			ys[i] = x % v
		}
		k := 1 + int(rng.Uint64()%10)
		if _, got := topK(ys, v, k); !slices.Equal(got, topKSort(ys, k)) {
			bad[1]++
		}
		if _, g := hIndex(ys); g != hIndexSort(ys) {
			bad[2]++
		}
	}
	fmt.Printf("\nbrute-force check: %d cases per problem (%d narrow), %d keys each\n", cases, cases/2, elems)
	for i, name := range []string{"maximum gap", "top-K frequent", "H-index"} {
		fmt.Printf("   %-15s mismatches: %d\n", name, bad[i])
	}

	// "Linear", counted on one big input: n reads plus n-sized structures.
	const big = 1_000_000
	zs := make([]int, big)
	for i := range zs {
		zs[i] = int(rng.Uint64() % big)
	}
	bs, _ = maxGapBuckets(zs)
	byFreq, _ = topK(zs, big, 10)
	cnt, h = hIndex(zs)
	fmt.Printf("\nn = %d keys in [0, n): each problem reads the n keys once, plus\n", big)
	fmt.Printf("   %-15s %9d buckets (min/max pairs)\n", "maximum gap", len(bs))
	fmt.Printf("   %-15s %9d frequency counters + %d frequency buckets\n", "top-K frequent", big, len(byFreq))
	fmt.Printf("   %-15s %9d count slots, h = %d found after scanning %d of them\n", "H-index", len(cnt), h, big-h+1)
}
```

**Output:**

```
1. maximum gap
   xs = [1 3 4 20 9 10 2], n = 7, R = 19, bucket width R/(n-1) = 3.17
   bucket 0 [ 1.00,  4.17)  min  1  max  4
   bucket 1 [ 4.17,  7.33)  empty
   bucket 2 [ 7.33, 10.50)  min  9  max 10
   bucket 3 [10.50, 13.67)  empty
   bucket 4 [13.67, 16.83)  empty
   bucket 5 [16.83, 20.00)  empty
   bucket 6 [20.00, 23.17)  min 20  max 20
   max gap = 10 (sorted: [1 2 3 4 9 10 20])

2. top-K frequent (higher frequency first, then smaller value)
   xs = [4 1 2 2 3 3 3 4 4 4 1 5 2 0], K = 3
   frequency 4: [4]
   frequency 3: [2 3]
   frequency 2: [1]
   frequency 1: [0 5]
   top 3 = [4 2 3]  (2 and 3 tie at 3; the smaller value goes first)

3. H-index
   citations = [3 0 6 1 5 25], n = 6: cap at n, 7 count slots
   cnt[0..n] = [1 1 0 1 0 1 2]  (25 counted as 6)
   h-index = 3

brute-force check: 20000 cases per problem (10000 narrow), 2006995 keys each
   maximum gap     mismatches: 0
   top-K frequent  mismatches: 0
   H-index         mismatches: 0

n = 1000000 keys in [0, n): each problem reads the n keys once, plus
   maximum gap       1000000 buckets (min/max pairs)
   top-K frequent    1000000 frequency counters + 1000001 frequency buckets
   H-index           1000001 count slots, h = 500474 found after scanning 499527 of them
```

**Rust:**

```rust
use std::cmp::Reverse;
use std::collections::HashMap;
use std::fmt::Display;

// Three interview problems whose Theta(n) solution IS a counting or bucket sort
// that stops early: it builds the buckets and reads the answer off them,
// without ever finishing the sort.

/// Port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state): with the same
/// seed (1, 2) it produces Go's exact stream, so all 20,000 cases match Go's.
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
        // DXSM "double xorshift multiply"
        let mut hi = self.hi;
        hi ^= hi >> 32;
        hi = hi.wrapping_mul(0xda942042e4dd58b5);
        hi ^= hi >> 48;
        hi.wrapping_mul(self.lo | 1)
    }
}

/// Go's %v of a slice: [1 2 3]
fn list<T: Display>(xs: &[T]) -> String {
    let parts: Vec<String> = xs.iter().map(|x| x.to_string()).collect();
    format!("[{}]", parts.join(" "))
}

// ---- 1. Maximum gap: largest difference between neighbours in sorted order.
//
// Let R = max - min. The n-1 gaps of the sorted order add up to R, so the
// largest is at least R/(n-1) (pigeonhole). Cut [min, max] into buckets of
// width R/(n-1): bucket floor((x-min)*(n-1)/R), n of them (the last one holds
// only max). Two keys in the same bucket differ by LESS than R/(n-1), so the
// maximum gap is never inside a bucket -- it runs from one bucket's max to the
// next non-empty bucket's min. Each bucket only needs its min and max.
//
// An empty bucket is None -- no `used` flag, no sentinel min/max values.

fn max_gap_buckets(xs: &[u64]) -> (Vec<Option<(u64, u64)>>, u64) {
    let n = xs.len() as u64;
    // min()/max() return Option: None for an empty slice.
    let (Some(&lo), Some(&hi)) = (xs.iter().min(), xs.iter().max()) else {
        return (Vec::new(), 0);
    };
    if n < 2 || lo == hi {
        return (Vec::new(), 0);
    }
    let mut bs = vec![None; n as usize];
    for &x in xs {
        let i = ((x - lo) * (n - 1) / (hi - lo)) as usize; // integer floor: exact
        bs[i] = Some(match bs[i] {
            None => (x, x),
            Some((mn, mx)) => (x.min(mn), x.max(mx)),
        });
    }
    // Bucket 0 always holds lo; walk only the non-empty ones.
    let mut used = bs.iter().flatten();
    let mut prev = used.next().expect("bucket 0 holds min").1;
    let mut gap = 0;
    for &(mn, mx) in used {
        gap = gap.max(mn - prev);
        prev = mx;
    }
    (bs, gap)
}

fn max_gap_sort(xs: &[u64]) -> u64 {
    let mut s = xs.to_vec();
    s.sort_unstable();
    s.windows(2).map(|w| w[1] - w[0]).max().unwrap_or(0)
}

// ---- 2. Top-K frequent values in [0, v): count, then bucket BY FREQUENCY.
//
// A frequency is at most n, so frequencies 1..n are bucket indices. Walking the
// values in increasing order puts smaller values first inside each bucket, so
// reading the buckets from n down gives the tie rule for free:
// higher frequency first, then smaller value.

fn top_k(xs: &[usize], v: usize, k: usize) -> (Vec<Vec<usize>>, Vec<usize>) {
    let mut freq = vec![0usize; v];
    for &x in xs {
        freq[x] += 1;
    }
    let mut by_freq = vec![Vec::new(); xs.len() + 1];
    for (val, &f) in freq.iter().enumerate() {
        if f > 0 {
            by_freq[f].push(val);
        }
    }
    // Buckets from high to low frequency, flattened, first k: one iterator chain.
    let out = by_freq.iter().rev().flatten().copied().take(k).collect();
    (by_freq, out)
}

fn top_k_sort(xs: &[usize], k: usize) -> Vec<usize> {
    let mut freq: HashMap<usize, usize> = HashMap::new();
    for &x in xs {
        *freq.entry(x).or_default() += 1;
    }
    let mut vals: Vec<usize> = freq.keys().copied().collect(); // random order: sort next
    vals.sort_unstable_by_key(|&v| (Reverse(freq[&v]), v));
    vals.truncate(k);
    vals
}

// ---- 3. H-index: the largest h with at least h papers cited >= h times.
//
// h can never exceed n, so a paper with 500 citations counts the same as one
// with n: cap at n and a count array of n+1 slots suffices.

fn h_index(cites: &[usize]) -> (Vec<usize>, usize) {
    let n = cites.len();
    let mut cnt = vec![0; n + 1];
    for &c in cites {
        cnt[c.min(n)] += 1;
    }
    let mut at_least = 0; // papers with >= h citations
    let mut h = n;
    while h > 0 {
        at_least += cnt[h];
        if at_least >= h {
            break;
        }
        h -= 1;
    }
    (cnt, h)
}

fn h_index_sort(cites: &[usize]) -> usize {
    let mut s = cites.to_vec();
    s.sort_unstable_by(|a, b| b.cmp(a)); // descending
    s.iter().enumerate().take_while(|&(i, &c)| c > i).count()
}

fn main() {
    println!("1. maximum gap");
    let gx: [u64; 7] = [1, 3, 4, 20, 9, 10, 2];
    let (bs, gap) = max_gap_buckets(&gx);
    let n = gx.len() as u64;
    let lo = *gx.iter().min().unwrap();
    let r = gx.iter().max().unwrap() - lo;
    println!(
        "   xs = {}, n = {n}, R = {r}, bucket width R/(n-1) = {:.2}",
        list(&gx),
        r as f64 / (n - 1) as f64
    );
    for (i, b) in bs.iter().enumerate() {
        let i = i as u64;
        let from = (lo * (n - 1) + i * r) as f64 / (n - 1) as f64; // exact integers, one division
        let to = (lo * (n - 1) + (i + 1) * r) as f64 / (n - 1) as f64;
        match b {
            Some((mn, mx)) => {
                println!("   bucket {i} [{from:5.2}, {to:5.2})  min {mn:2}  max {mx:2}")
            }
            None => println!("   bucket {i} [{from:5.2}, {to:5.2})  empty"),
        }
    }
    let mut sorted = gx;
    sorted.sort_unstable();
    println!("   max gap = {gap} (sorted: {})", list(&sorted));

    println!("\n2. top-K frequent (higher frequency first, then smaller value)");
    let tx = [4, 1, 2, 2, 3, 3, 3, 4, 4, 4, 1, 5, 2, 0];
    let (by_freq, top) = top_k(&tx, 6, 3);
    println!("   xs = {}, K = 3", list(&tx));
    for (f, vals) in by_freq.iter().enumerate().rev() {
        if !vals.is_empty() {
            println!("   frequency {f}: {}", list(vals));
        }
    }
    println!(
        "   top 3 = {}  (2 and 3 tie at 3; the smaller value goes first)",
        list(&top)
    );

    println!("\n3. H-index");
    let hx = [3, 0, 6, 1, 5, 25];
    let (cnt, h) = h_index(&hx);
    println!(
        "   citations = {}, n = {}: cap at n, {} count slots",
        list(&hx),
        hx.len(),
        cnt.len()
    );
    println!(
        "   cnt[0..n] = {}  (25 counted as {})",
        list(&cnt),
        hx.len()
    );
    println!("   h-index = {h}");

    // Proof by brute force: 20,000 random cases per problem, half of them on a
    // narrow value range (0..7) so duplicates, equal values and ties happen.
    const CASES: usize = 20_000;
    let mut rng = Pcg::new(1, 2);
    let mut bad = [0; 3];
    let mut elems = 0;
    for c in 0..CASES {
        let m = (rng.next_u64() % 201) as usize; // lengths 0..200
        let span: u64 = if c % 2 == 1 { 8 } else { 1_000_000_000 };
        let xs: Vec<u64> = (0..m).map(|_| rng.next_u64() % span).collect();
        elems += m;
        if max_gap_buckets(&xs).1 != max_gap_sort(&xs) {
            bad[0] += 1;
        }
        // top-K needs a bounded value range: reuse the keys mod 1000 (or 8).
        let v = span.min(1000);
        let ys: Vec<usize> = xs.iter().map(|&x| (x % v) as usize).collect();
        let k = 1 + (rng.next_u64() % 10) as usize;
        if top_k(&ys, v as usize, k).1 != top_k_sort(&ys, k) {
            bad[1] += 1;
        }
        if h_index(&ys).1 != h_index_sort(&ys) {
            bad[2] += 1;
        }
    }
    println!(
        "\nbrute-force check: {CASES} cases per problem ({} narrow), {elems} keys each",
        CASES / 2
    );
    for (name, b) in ["maximum gap", "top-K frequent", "H-index"].iter().zip(bad) {
        println!("   {name:<15} mismatches: {b}");
    }

    // "Linear", counted on one big input: n reads plus n-sized structures.
    const BIG: usize = 1_000_000;
    let zs: Vec<usize> = (0..BIG)
        .map(|_| (rng.next_u64() % BIG as u64) as usize)
        .collect();
    let zs64: Vec<u64> = zs.iter().map(|&z| z as u64).collect();
    let (bs, _) = max_gap_buckets(&zs64);
    let (by_freq, _) = top_k(&zs, BIG, 10);
    let (cnt, h) = h_index(&zs);
    println!("\nn = {BIG} keys in [0, n): each problem reads the n keys once, plus");
    println!(
        "   {:<15} {:>9} buckets (min/max pairs)",
        "maximum gap",
        bs.len()
    );
    println!(
        "   {:<15} {:>9} frequency counters + {} frequency buckets",
        "top-K frequent",
        BIG,
        by_freq.len()
    );
    println!(
        "   {:<15} {:>9} count slots, h = {h} found after scanning {} of them",
        "H-index",
        cnt.len(),
        BIG - h + 1
    );
}
```

**Rust output:**

```
1. maximum gap
   xs = [1 3 4 20 9 10 2], n = 7, R = 19, bucket width R/(n-1) = 3.17
   bucket 0 [ 1.00,  4.17)  min  1  max  4
   bucket 1 [ 4.17,  7.33)  empty
   bucket 2 [ 7.33, 10.50)  min  9  max 10
   bucket 3 [10.50, 13.67)  empty
   bucket 4 [13.67, 16.83)  empty
   bucket 5 [16.83, 20.00)  empty
   bucket 6 [20.00, 23.17)  min 20  max 20
   max gap = 10 (sorted: [1 2 3 4 9 10 20])

2. top-K frequent (higher frequency first, then smaller value)
   xs = [4 1 2 2 3 3 3 4 4 4 1 5 2 0], K = 3
   frequency 4: [4]
   frequency 3: [2 3]
   frequency 2: [1]
   frequency 1: [0 5]
   top 3 = [4 2 3]  (2 and 3 tie at 3; the smaller value goes first)

3. H-index
   citations = [3 0 6 1 5 25], n = 6: cap at n, 7 count slots
   cnt[0..n] = [1 1 0 1 0 1 2]  (25 counted as 6)
   h-index = 3

brute-force check: 20000 cases per problem (10000 narrow), 2006995 keys each
   maximum gap     mismatches: 0
   top-K frequent  mismatches: 0
   H-index         mismatches: 0

n = 1000000 keys in [0, n): each problem reads the n keys once, plus
   maximum gap       1000000 buckets (min/max pairs)
   top-K frequent    1000000 frequency counters + 1000001 frequency buckets
   H-index           1000001 count slots, h = 500474 found after scanning 499527 of them
```

> **Go vs Rust:** Output is identical to Go's character for character, including the 2,006,995 random keys of the brute-force check (same PCG-DXSM stream, `Uint64() % m` on both sides). The shapes differ where Rust has a type for it: an empty max-gap bucket is `None` instead of Go's `used` flag, `xs.iter().min()` returns `Option` so the empty slice is a `let ... else`, and top-K's answer is one chain `by_freq.iter().rev().flatten().take(k)` instead of two nested loops.

**Complexity:** each Θ(n) time and Θ(n) space (top-K: Θ(n + v) for values in [0, v)) — one read of the keys plus n-slot structures, measured as 10⁶ buckets / 10⁶ + 1 slots at n = 10⁶ · vs Θ(n log n) by sorting · 0 mismatches against the sort-based brute force in 3 × 20,000 cases.

---

> Next tier: [🔴 hard](3-hard.md) — the measured crossover against `slices.Sort`, real keys,
> strings with shared prefixes, radix sort in place, and the capstone.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
