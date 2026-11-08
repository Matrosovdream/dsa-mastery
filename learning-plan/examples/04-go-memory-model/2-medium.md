# Step 04 — Go's Memory Model · 🟡 Medium

Examples **7–11**: cache lines, access order, and the three taxes on a pointer.

> ⚠️ **Sample output.** This whole tier measures time on real hardware — an Apple M4 with Go 1.26.3
> and a **128-byte** cache line. On a typical x86 machine the line is 64 bytes and example 7's
> plateau arrives at stride 8 instead of 16. The *shapes* are what must match, not the digits.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

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

## 7. Measuring your cache line from Go

`🟡 medium` · *The stride cliff*

The CPU never loads one `int`; it loads a **cache line** containing that int and its neighbours. Walk
an array with a growing stride and the cost *per touch* climbs until each touch needs its own line —
then it flattens. **Where it flattens tells you the line size.**

Note the `sink`. Without it the compiler deletes the loads and every stride reports an identical
0.23 ns per touch — which would be faster than memory bandwidth allows. (Lesson 01, example 12.)

**Steps:**

1. Sum every k-th element of a 32 MB array for k = 1…128.
2. Divide by the number of touches, not by n.
3. Find where the per-touch cost stops climbing, and multiply by 8 bytes.

```go
package main

import (
	"fmt"
	"testing"
)

// sink consumes the result. Without it the compiler deletes the loads and every
// stride measures the same -- an impossible 0.23 ns per touch of a 32 MB array.
// (Lesson 01, example 12: a benchmark that discards its result measures nothing.)
var sink int

// The CPU never loads one int. It loads a CACHE LINE -- a fixed block of bytes
// containing that int and its neighbours (64 on most x86, 128 on Apple silicon).
//
// sumStride touches every k-th element, so it does n/k additions. If memory were
// flat, the cost PER TOUCH would be constant. Instead it climbs until each touch
// needs its own cache line, then flattens at the price of one line fetch.
func sumStride(xs []int, stride int) int {
	total := 0
	for i := 0; i < len(xs); i += stride {
		total += xs[i]
	}
	return total
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
	const n = 1 << 22 // 4M ints = 32 MB, far larger than any cache
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i
	}

	fmt.Printf("summing every k-th element of %d ints (%d MB)\n\n", n, n*8/(1<<20))
	fmt.Printf("%8s %10s %12s %14s %16s\n",
		"stride", "bytes", "touches", "total ns", "ns per touch")

	var first float64
	for _, stride := range []int{1, 2, 4, 8, 16, 32, 64, 128} {
		touches := n / stride
		total := nsPerOp(func() { sink += sumStride(xs, stride) })
		perTouch := total / float64(touches)
		if first == 0 {
			first = perTouch
		}

		note := ""
		switch {
		case perTouch > 4*first:
			note = "  <- every touch is its own line"
		case perTouch > 2*first:
			note = "  <- neighbours mostly wasted"
		}
		fmt.Printf("%8d %10d %12d %14.0f %16.2f%s\n",
			stride, stride*8, touches, total, perTouch, note)
	}

	fmt.Println()
	fmt.Println("at stride 1 you pay for a line once and get its other 15 ints nearly free.")
	fmt.Println("As the stride grows, fewer of those neighbours are used -- until each touch")
	fmt.Println("needs a line of its own and the per-touch cost stops climbing.")
	fmt.Println()
	fmt.Println("WHERE it stops tells you the line size: the jump lands at stride 16,")
	fmt.Println("and 16 * 8 bytes = 128 bytes. That is this machine's cache line, measured")
	fmt.Println("from Go with no system calls. (On a 64-byte machine it lands at stride 8.)")
	fmt.Println()
	fmt.Println("the last row usually dips a little. At stride 128 only 32768 lines are")
	fmt.Println("touched -- 4 MB of distinct lines, small enough to start fitting in cache")
	fmt.Println("again. The plateau is the signal; the tail is a second-order effect.")
	fmt.Println()
	fmt.Println("consequence: an algorithm's cost is not its operation count, it is its")
	fmt.Println("LINE count. Touching 8 neighbours costs about the same as touching 1.")
}
```

**Sample output:**

```
summing every k-th element of 4194304 ints (32 MB)

  stride      bytes      touches       total ns     ns per touch
       1          8      4194304        1137326             0.27
       2         16      2097152         580937             0.28
       4         32      1048576         455940             0.43
       8         64       524288         444651             0.85  <- neighbours mostly wasted
      16        128       262144         467095             1.78  <- every touch is its own line
      32        256       131072         238921             1.82  <- every touch is its own line
      64        512        65536         118824             1.81  <- every touch is its own line
     128       1024        32768          41582             1.27  <- every touch is its own line

at stride 1 you pay for a line once and get its other 15 ints nearly free.
As the stride grows, fewer of those neighbours are used -- until each touch
needs a line of its own and the per-touch cost stops climbing.

WHERE it stops tells you the line size: the jump lands at stride 16,
and 16 * 8 bytes = 128 bytes. That is this machine's cache line, measured
from Go with no system calls. (On a 64-byte machine it lands at stride 8.)

the last row usually dips a little. At stride 128 only 32768 lines are
touched -- 4 MB of distinct lines, small enough to start fitting in cache
again. The plateau is the signal; the tail is a second-order effect.

consequence: an algorithm's cost is not its operation count, it is its
LINE count. Touching 8 neighbours costs about the same as touching 1.
```

The plateau lands at **stride 16 = 128 bytes**, which is this machine's line size — derived from Go
alone, no system calls. On a 64-byte machine it lands at stride 8.

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// Go's testing.Benchmark / b.Loop becomes this tiny harness: run the closure
// until 200 ms have elapsed and report ns per call.
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

// The CPU never loads one i64. It loads a CACHE LINE -- a fixed block of bytes
// containing that i64 and its neighbours (64 on most x86, 128 on Apple silicon).
//
// sum_stride touches every k-th element, so it does n/k additions. If memory
// were flat, the cost PER TOUCH would be constant. Instead it climbs until each
// touch needs its own cache line, then flattens at the price of one line fetch.
fn sum_stride(xs: &[i64], stride: usize) -> i64 {
    xs.iter().step_by(stride).sum()
}

fn main() {
    const N: usize = 1 << 22; // 4M ints = 32 MB, far larger than any cache
    let xs: Vec<i64> = (0..N as i64).collect();

    println!(
        "summing every k-th element of {N} ints ({} MB)\n",
        N * 8 / (1 << 20)
    );
    println!(
        "{:>8} {:>10} {:>12} {:>14} {:>16}",
        "stride", "bytes", "touches", "total ns", "ns per touch"
    );

    // black_box is Go's `sink`: it hides the inputs from the optimiser and
    // forces the result to be "used". Without it the sum is pure and unused,
    // rustc deletes the loop, and every stride reports 0 ns (checked: it did).
    let mut rows = Vec::new(); // (stride, ns per touch, total ns)
    for stride in [1usize, 2, 4, 8, 16, 32, 64, 128] {
        let touches = N / stride;
        let total = ns_per_op(|| {
            black_box(sum_stride(black_box(&xs), black_box(stride)));
        });
        let per_touch = total / touches as f64;
        let first = rows.first().map_or(per_touch, |&(_, p, _)| p);

        let note = if per_touch > 4.0 * first {
            "  <- every touch is its own line"
        } else if per_touch > 2.0 * first {
            "  <- neighbours mostly wasted"
        } else {
            ""
        };
        println!(
            "{stride:>8} {:>10} {touches:>12} {total:>14.0} {per_touch:>16.2}{note}",
            stride * 8
        );
        rows.push((stride, per_touch, total));
    }

    // Read the plateau off the measurements: the first stride whose cost is
    // within 20% of the most expensive row.
    let peak = rows.iter().map(|&(_, p, _)| p).fold(0.0, f64::max);
    let (plateau, plateau_cost, plateau_total) = rows
        .iter()
        .copied()
        .find(|&(_, p, _)| p >= 0.8 * peak)
        .expect("the peak row itself qualifies");
    let (_, first, first_total) = rows[0];
    let (_, at_64, _) = rows[rows.len() - 2];
    let (_, at_128, _) = rows[rows.len() - 1];

    println!();
    println!("at stride 1 a line is paid for once and every int in it is used:");
    println!("{first:.2} ns per touch. At stride {plateau} each touch needs a new line:");
    println!(
        "{plateau_cost:.2} ns, {:.1}x more. Between the two, fewer neighbours are used.",
        plateau_cost / first
    );
    println!();
    println!("WHERE it stops climbing tells you the line size: the jump lands at stride");
    println!(
        "{plateau}, and {plateau} * 8 bytes = {} bytes. That is this machine's cache line,",
        plateau * 8
    );
    println!(
        "measured from Rust with no system calls. (On a 64-byte machine it lands at stride 8.)"
    );
    println!();
    println!(
        "the last row vs the one before it: {at_128:.2} vs {at_64:.2} ns per touch ({:.0}%).",
        100.0 * at_128 / at_64
    );
    println!("At stride 128 only 32768 lines are touched -- 4 MB of distinct lines, small");
    println!("enough to start fitting in cache again. The plateau is the signal; any");
    println!("difference in the tail is a second-order effect.");
    println!();
    println!("consequence: an algorithm's cost is not its operation count, it is its");
    println!(
        "LINE count. Stride 1 does {N} additions in {first_total:.0} ns; stride {plateau} does {plateau}x fewer"
    );
    println!(
        "({}) in {plateau_total:.0} ns -- only {:.1}x less time, because it still touches every line.",
        N / plateau,
        first_total / plateau_total
    );
}
```

**Rust sample output:**

```
summing every k-th element of 4194304 ints (32 MB)

  stride      bytes      touches       total ns     ns per touch
       1          8      4194304        1704470             0.41
       2         16      2097152         891427             0.43
       4         32      1048576         698436             0.67
       8         64       524288         510761             0.97  <- neighbours mostly wasted
      16        128       262144         615835             2.35  <- every touch is its own line
      32        256       131072         311704             2.38  <- every touch is its own line
      64        512        65536         127999             1.95  <- every touch is its own line
     128       1024        32768          61165             1.87  <- every touch is its own line

at stride 1 a line is paid for once and every int in it is used:
0.41 ns per touch. At stride 16 each touch needs a new line:
2.35 ns, 5.8x more. Between the two, fewer neighbours are used.

WHERE it stops climbing tells you the line size: the jump lands at stride
16, and 16 * 8 bytes = 128 bytes. That is this machine's cache line,
measured from Rust with no system calls. (On a 64-byte machine it lands at stride 8.)

the last row vs the one before it: 1.87 vs 1.95 ns per touch (96%).
At stride 128 only 32768 lines are touched -- 4 MB of distinct lines, small
enough to start fitting in cache again. The plateau is the signal; any
difference in the tail is a second-order effect.

consequence: an algorithm's cost is not its operation count, it is its
LINE count. Stride 1 does 4194304 additions in 1704470 ns; stride 16 does 16x fewer
(262144) in 615835 ns -- only 2.8x less time, because it still touches every line.
```

> **Go vs Rust:** Pure hardware, so the Rust numbers land where Go's did on the same M4: per-touch cost flat at strides 1 and 2, climbing through 4 and 8, then a plateau from stride 16 (128 bytes) in every run, 5.5x to 7x the stride-1 cost over six runs (Go: 6.6x). `sysctl -n hw.cachelinesize` reports 128 on this machine, so the plateau is the real line size. What differs is plumbing: `black_box` replaces the package-level `sink` (without it rustc deletes the loop and every stride prints 0 ns, which was tried), `step_by` replaces the index loop, and the plateau stride is read off the data instead of hard-coded. Go's 'the last row usually dips' held in most Rust runs (stride 128 below stride 64) but not all, so the Rust text prints the two numbers instead of asserting a dip.

**Complexity:** every row does Θ(n/stride) additions · the cost is governed by **lines touched**, not elements touched, and those stop being the same thing at stride 16

---

## 8. The same reads, in a different order

`🟡 medium` · *Locality*

`sumIndexed` reads exactly `len(idx)` elements. Hand it `0,1,2,…` and it walks memory; hand it a
shuffle and it hops. Identical instruction count, identical Θ(n), identical data — and up to **7.1×**
apart.

**Steps:**

1. Build one index slice in order and one shuffled.
2. Run both at four working-set sizes, from L1-resident to far larger than any cache.
3. Watch the penalty appear only once the data stops fitting.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"testing"
)

// Without the sink the compiler removes the loads and both orders measure
// identically -- see lesson 01, example 12.
var sink int

// sumIndexed reads exactly len(idx) elements of xs, in whatever order idx says.
// Feed it 0,1,2,... and it walks memory. Feed it a shuffle and it hops.
// Same instruction count, same Big-O, same data. Only the ORDER changes.
func sumIndexed(xs, idx []int) int {
	total := 0
	for _, i := range idx {
		total += xs[i]
	}
	return total
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
	rng := rand.New(rand.NewPCG(1, 2))

	fmt.Println("identical reads, identical count -- only the ORDER differs:")
	fmt.Println()
	fmt.Printf("%10s %12s %14s %14s %14s %10s\n",
		"n", "size", "in order ns", "shuffled ns", "ns/read shuf", "penalty")

	for _, n := range []int{1 << 10, 1 << 14, 1 << 18, 1 << 22} {
		xs := make([]int, n)
		inOrder := make([]int, n)
		for i := range xs {
			xs[i] = i
			inOrder[i] = i
		}
		shuffled := make([]int, n)
		copy(shuffled, inOrder)
		rng.Shuffle(n, func(i, j int) { shuffled[i], shuffled[j] = shuffled[j], shuffled[i] })

		seq := nsPerOp(func() { sink += sumIndexed(xs, inOrder) })
		rnd := nsPerOp(func() { sink += sumIndexed(xs, shuffled) })

		size := fmt.Sprintf("%d KB", n*8/1024)
		fmt.Printf("%10d %12s %14.0f %14.0f %14.2f %9.1fx\n",
			n, size, seq, rnd, rnd/float64(n), rnd/seq)
	}

	fmt.Println()
	fmt.Println("the penalty grows with the working set:")
	fmt.Println("  8 KB    -- fits in L1, order is irrelevant")
	fmt.Println("  2 MB    -- spills out of L2, misses start to bite")
	fmt.Println("  32 MB   -- every shuffled read is a miss")
	fmt.Println()
	fmt.Println("prefetching is why in-order is so cheap: the CPU spots the pattern and")
	fmt.Println("fetches ahead. A random order defeats it completely, and you pay full")
	fmt.Println("latency on every single read.")
	fmt.Println()
	fmt.Println("this one effect underlies most of the plan's 'the slice wins' results:")
	fmt.Println("  - a linked list is a shuffled walk BY CONSTRUCTION (example 9)")
	fmt.Println("  - a tree descent jumps to an unrelated line at every level (example 14)")
	fmt.Println("  - it is why an O(n) scan beats an O(log n) search at small n")
}
```

**Sample output:**

```
identical reads, identical count -- only the ORDER differs:

         n         size    in order ns    shuffled ns   ns/read shuf    penalty
      1024         8 KB            286            289           0.28       1.0x
     16384       128 KB           4585           5566           0.34       1.2x
    262144      2048 KB          92611         146144           0.56       1.6x
   4194304     32768 KB        1233507        8743736           2.08       7.1x

the penalty grows with the working set:
  8 KB    -- fits in L1, order is irrelevant
  2 MB    -- spills out of L2, misses start to bite
  32 MB   -- every shuffled read is a miss

prefetching is why in-order is so cheap: the CPU spots the pattern and
fetches ahead. A random order defeats it completely, and you pay full
latency on every single read.

this one effect underlies most of the plan's 'the slice wins' results:
  - a linked list is a shuffled walk BY CONSTRUCTION (example 9)
  - a tree descent jumps to an unrelated line at every level (example 14)
  - it is why an O(n) scan beats an O(log n) search at small n
```

At 8 KB the order is irrelevant — everything is in L1. The penalty grows with the working set until,
at 32 MB, every shuffled read is a miss. **Prefetching** is the mechanism: the CPU spots a sequential
pattern and fetches ahead; a random order defeats it entirely.

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// Go's testing.Benchmark / b.Loop becomes this tiny harness: run the closure
// until 200 ms have elapsed and report ns per call.
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

// Go's math/rand: seeded splitmix64 and a Fisher-Yates shuffle. Seeded, but not
// Go's PCG stream, so the shuffles (and the timings) are not Go's.
struct Rng(u64);

impl Rng {
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
    fn shuffle<T>(&mut self, xs: &mut [T]) {
        for i in (1..xs.len()).rev() {
            xs.swap(i, self.below(i as u64 + 1) as usize);
        }
    }
}

// sum_indexed reads exactly idx.len() elements of xs, in whatever order idx says.
// Feed it 0,1,2,... and it walks memory. Feed it a shuffle and it hops.
// Same instruction count, same Big-O, same data. Only the ORDER changes.
// (The indexing is bounds-checked, as Go's is; both orders pay that equally.)
fn sum_indexed(xs: &[i64], idx: &[usize]) -> i64 {
    idx.iter().map(|&i| xs[i]).sum()
}

fn main() {
    let mut rng = Rng(1);

    println!("identical reads, identical count -- only the ORDER differs:\n");
    println!(
        "{:>10} {:>12} {:>14} {:>14} {:>14} {:>10}",
        "n", "size", "in order ns", "shuffled ns", "ns/read shuf", "penalty"
    );

    let mut rows = Vec::new(); // (size in KB, seq ns/read, shuffled ns/read, penalty)
    for n in [1usize << 10, 1 << 14, 1 << 18, 1 << 22] {
        let xs: Vec<i64> = (0..n as i64).collect();
        let in_order: Vec<usize> = (0..n).collect();
        let mut shuffled = in_order.clone();
        rng.shuffle(&mut shuffled);

        // black_box is Go's `sink`: without it rustc may delete the loads.
        let seq = ns_per_op(|| {
            black_box(sum_indexed(black_box(&xs), black_box(&in_order)));
        });
        let rnd = ns_per_op(|| {
            black_box(sum_indexed(black_box(&xs), black_box(&shuffled)));
        });

        let kb = n * 8 / 1024;
        let size = format!("{kb} KB");
        println!(
            "{n:>10} {size:>12} {seq:>14.0} {rnd:>14.0} {:>14.2} {:>9.1}x",
            rnd / n as f64,
            rnd / seq
        );
        rows.push((kb, seq / n as f64, rnd / n as f64, rnd / seq));
    }

    let (kb_small, seq_small, rnd_small, pen_small) = rows[0];
    let (kb_big, seq_big, rnd_big, pen_big) = rows[rows.len() - 1];
    println!();
    println!(
        "the penalty grows with the working set: {pen_small:.1}x at {kb_small} KB, {pen_big:.1}x at {kb_big} KB. (The size is xs;"
    );
    println!("idx is read too, so twice that is touched.)");
    println!();
    println!("ns per read, smallest size -> largest size:");
    println!(
        "  in order  {seq_small:.2} -> {seq_big:.2}   ({:.1}x)",
        seq_big / seq_small
    );
    println!(
        "  shuffled  {rnd_small:.2} -> {rnd_big:.2}   ({:.1}x)",
        rnd_big / rnd_small
    );
    println!("the sequential walk barely notices the data growing; the shuffled one does.");
    println!();
    println!("the usual explanation is hardware prefetching: the CPU spots a sequential");
    println!("pattern and fetches ahead, and a random order defeats it, so every read pays");
    println!("latency. This program shows the effect, not the prefetcher; that part is");
    println!("the standard account, not something measured here.");
    println!();
    println!("this one effect underlies most of the plan's 'the slice wins' results:");
    println!("  - a linked list is a shuffled walk BY CONSTRUCTION");
    println!("  - a tree descent jumps to an unrelated line at every level");
    println!("  - it is why an O(n) scan can beat an O(log n) search at small n");
}
```

**Rust sample output:**

```
identical reads, identical count -- only the ORDER differs:

         n         size    in order ns    shuffled ns   ns/read shuf    penalty
      1024         8 KB            276            303           0.30       1.1x
     16384       128 KB           4738           6393           0.39       1.3x
    262144      2048 KB          91210         165810           0.63       1.8x
   4194304     32768 KB        1265394        9727475           2.32       7.7x

the penalty grows with the working set: 1.1x at 8 KB, 7.7x at 32768 KB. (The size is xs;
idx is read too, so twice that is touched.)

ns per read, smallest size -> largest size:
  in order  0.27 -> 0.30   (1.1x)
  shuffled  0.30 -> 2.32   (7.8x)
the sequential walk barely notices the data growing; the shuffled one does.

the usual explanation is hardware prefetching: the CPU spots a sequential
pattern and fetches ahead, and a random order defeats it, so every read pays
latency. This program shows the effect, not the prefetcher; that part is
the standard account, not something measured here.

this one effect underlies most of the plan's 'the slice wins' results:
  - a linked list is a shuffled walk BY CONSTRUCTION
  - a tree descent jumps to an unrelated line at every level
  - it is why an O(n) scan can beat an O(log n) search at small n
```

> **Go vs Rust:** Rust agrees with Go here. Over a dozen runs the penalty was 0.9-1.3x at 8 KB and at 128 KB (below 1.0x is timing noise), 1.6-1.9x at 2 MB, and never below 7x at 32 MB (7-9x on a quiet machine, 11-18x when other jobs were competing for memory; Go: 1.0x, 1.2x, 1.6x, 7.1x). The in-order walk cost 0.27-0.38 ns per read at every size (Go: 0.28 and 0.29), while the shuffled walk went from about 0.3 to 2-5 ns. `black_box` stands in for Go's `sink`, and `xs[i]` is bounds-checked in both languages. The shuffle is a seeded splitmix64 Fisher-Yates, not Go's PCG, so the permutation differs but not the effect. Prefetching is the textbook explanation for it and neither program measures it.

**Complexity:** both are Θ(n) with identical operation counts · the 7.1× is entirely locality, and it is invisible in every complexity table

---

## 9. The three taxes on `[]*T`

`🟡 medium` · *Pointers cost more than a dereference*

Lesson 01 showed this comparison; here is the full accounting. A pointer slice pays on **memory**,
**locality** and **GC tracing** — and the third column below makes the mechanism concrete.

**Steps:**

1. Build 1M points as values, and as separately-allocated pointers in shuffled order.
2. Measure each in its own scope, so the GC timing reflects only that structure.
3. Compare sum time, GC cycle time, and live heap objects.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"testing"
	"time"
)

var sink int

type point struct{ x, y int }

func sumValues(ps []point) int {
	total := 0
	for i := range ps {
		total += ps[i].x + ps[i].y
	}
	return total
}

func sumPointers(ps []*point) int {
	total := 0
	for _, p := range ps {
		total += p.x + p.y
	}
	return total
}

func nsPerOp(run func()) float64 {
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			run()
		}
	})
	return float64(res.T.Nanoseconds()) / float64(res.N)
}

// avgGC times a full collection. Whatever is REACHABLE when this runs is what
// the collector has to walk -- so each phase must measure in its own scope.
func avgGC(rounds int) time.Duration {
	runtime.GC() // warm up; do not let the first, larger cycle skew the average
	start := time.Now()
	for i := 0; i < rounds; i++ {
		runtime.GC()
	}
	return time.Since(start) / time.Duration(rounds)
}

func measureValues(n int) (float64, time.Duration, uint64) {
	values := make([]point, n)
	for i := range values {
		values[i] = point{x: i, y: i}
	}
	t := nsPerOp(func() { sink += sumValues(values) })
	gc := avgGC(10)
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	runtime.KeepAlive(values)
	return t, gc, ms.HeapObjects
}

func measurePointers(n int) (float64, time.Duration, uint64) {
	pointers := make([]*point, n)
	for i := range pointers {
		pointers[i] = &point{x: i, y: i}
	}
	// Shuffle: a structure built up over its lifetime is not laid out in
	// allocation order, and the prefetcher can no longer guess what is next.
	rng := rand.New(rand.NewPCG(3, 5))
	rng.Shuffle(n, func(i, j int) { pointers[i], pointers[j] = pointers[j], pointers[i] })

	t := nsPerOp(func() { sink += sumPointers(pointers) })
	gc := avgGC(10)
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	runtime.KeepAlive(pointers)
	return t, gc, ms.HeapObjects
}

func main() {
	const n = 1_000_000

	valT, valGC, valObjs := measureValues(n)
	runtime.GC() // drop phase 1 before phase 2 allocates
	ptrT, ptrGC, ptrObjs := measurePointers(n)

	fmt.Printf("summing %d points, held two ways\n\n", n)

	fmt.Printf("%-24s %14s %16s %16s\n", "", "sum (ns/op)", "GC cycle", "live heap objects")
	fmt.Printf("%-24s %14.0f %16v %16d\n", "[]point (contiguous)", valT, valGC.Round(time.Microsecond), valObjs)
	fmt.Printf("%-24s %14.0f %16v %16d\n", "[]*point (chased)", ptrT, ptrGC.Round(time.Microsecond), ptrObjs)
	fmt.Printf("%-24s %13.1fx %15.0fx %15.0fx\n", "ratio",
		ptrT/valT, float64(ptrGC)/float64(valGC), float64(ptrObjs)/float64(valObjs))

	fmt.Println()
	fmt.Println("the pointer version pays THREE separate taxes, none of them in the Big-O:")
	fmt.Println()
	fmt.Println("  1. memory     2x the bytes (example 6): the pointer AND the pointee")
	fmt.Println("  2. locality   each element is a separate object, so walking the slice")
	fmt.Println("                is a shuffled walk -- the example 8 penalty, by construction")
	fmt.Println("  3. GC         the collector must trace every one of those pointers on")
	fmt.Println("                every cycle. []point contains no pointers at all, so the")
	fmt.Println("                GC can skip the whole block (example 12)")
	fmt.Println()
	fmt.Println("both loops are Theta(n). One walks memory, the other chases it.")
	fmt.Println("In Go, choosing []T over []*T is an algorithmic decision, not a style one.")
	fmt.Println()
	fmt.Println("when you DO need []*T: elements are huge and copied often, elements are")
	fmt.Println("shared/mutated through several owners, or the type is polymorphic.")
}
```

**Sample output:**

```
summing 1000000 points, held two ways

                            sum (ns/op)         GC cycle live heap objects
[]point (contiguous)             311141             62µs              174
[]*point (chased)                758038          3.666ms          1000206
ratio                              2.4x              59x            5748x

the pointer version pays THREE separate taxes, none of them in the Big-O:

  1. memory     2x the bytes (example 6): the pointer AND the pointee
  2. locality   each element is a separate object, so walking the slice
                is a shuffled walk -- the example 8 penalty, by construction
  3. GC         the collector must trace every one of those pointers on
                every cycle. []point contains no pointers at all, so the
                GC can skip the whole block (example 12)

both loops are Theta(n). One walks memory, the other chases it.
In Go, choosing []T over []*T is an algorithmic decision, not a style one.

when you DO need []*T: elements are huge and copied often, elements are
shared/mutated through several owners, or the type is polymorphic.
```

**174 heap objects versus 1,000,206.** That ratio *is* the GC column: the collector has a million
pointers to trace in one case and effectively none in the other.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::hint::black_box;
use std::mem::size_of;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Go reads runtime.MemStats.HeapObjects after a forced GC. Rust has no GC to
// force, so a counting allocator keeps the same two numbers exactly: blocks
// (objects) and bytes currently live on the heap. A global allocator is an
// `unsafe` trait by definition; this one only does arithmetic and forwards
// every call to the system allocator.
struct Counting;
static BLOCKS: AtomicUsize = AtomicUsize::new(0);
static BYTES: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        BLOCKS.fetch_add(1, Relaxed);
        BYTES.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        BLOCKS.fetch_sub(1, Relaxed);
        BYTES.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        BYTES.fetch_add(new, Relaxed);
        BYTES.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (live heap blocks, live heap bytes) right now.
fn heap() -> (usize, usize) {
    (BLOCKS.load(Relaxed), BYTES.load(Relaxed))
}

// Go's testing.Benchmark / b.Loop becomes this tiny harness: run the closure
// until 200 ms have elapsed and report ns per call.
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

// Go's math/rand: seeded splitmix64 and a Fisher-Yates shuffle.
struct Rng(u64);

impl Rng {
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
    fn shuffle<T>(&mut self, xs: &mut [T]) {
        for i in (1..xs.len()).rev() {
            xs.swap(i, self.below(i as u64 + 1) as usize);
        }
    }
}

struct Point {
    x: i64,
    y: i64,
}

fn sum_values(ps: &[Point]) -> i64 {
    ps.iter().map(|p| p.x + p.y).sum()
}

fn sum_pointers(ps: &[Box<Point>]) -> i64 {
    ps.iter().map(|p| p.x + p.y).sum()
}

/// What one structure costs. Go's "GC cycle" column has no Rust counterpart (there
/// is no collector); the cost Rust does pay is the allocation and the free.
struct Row {
    build: Duration,
    sum_ns: f64,
    drop: Duration,
    blocks: usize,
    bytes: usize,
}

// The machine is shared and timings are noisy, so every figure is the best of
// 3 runs: the least disturbed one.
fn best_ns_per_op(mut f: impl FnMut()) -> f64 {
    (0..3)
        .map(|_| ns_per_op(&mut f))
        .fold(f64::INFINITY, f64::min)
}

// Builds a structure and drops it, three times, and keeps the fastest build and
// the fastest drop. `prepare` runs between the two and is not timed (it is the
// shuffle, below). Go's GC timing needs each structure in its own scope; here
// the drop is a statement we can time, and ownership ends it.
fn best_build_and_drop<T>(
    build: impl Fn() -> T,
    mut prepare: impl FnMut(&mut T),
) -> (Duration, Duration) {
    let (mut best_build, mut best_drop) = (Duration::MAX, Duration::MAX);
    for _ in 0..3 {
        let t = Instant::now();
        let mut data = build();
        best_build = best_build.min(t.elapsed());
        prepare(&mut data);
        let t = Instant::now();
        drop(data);
        best_drop = best_drop.min(t.elapsed());
    }
    (best_build, best_drop)
}

fn build_values(n: usize) -> Vec<Point> {
    (0..n as i64).map(|i| Point { x: i, y: i }).collect()
}

// clippy's `vec_box` lint says what this whole example says: a Vec<Box<T>> of a
// small Sized T is usually a mistake. Here the mistake is the thing measured.
#[allow(clippy::vec_box)]
fn build_pointers(n: usize) -> Vec<Box<Point>> {
    (0..n as i64)
        .map(|i| Box::new(Point { x: i, y: i }))
        .collect()
}

fn measure_values(n: usize) -> Row {
    let (build, drop) = best_build_and_drop(|| build_values(n), |_| {});

    let (blocks0, bytes0) = heap();
    let values = build_values(n);
    let (blocks1, bytes1) = heap();
    let sum_ns = best_ns_per_op(|| {
        black_box(sum_values(black_box(&values)));
    });
    Row {
        build,
        sum_ns,
        drop,
        blocks: blocks1 - blocks0,
        bytes: bytes1 - bytes0,
    }
}

/// Share of neighbouring boxes that sit exactly one Point apart in memory.
/// Reading an address is safe; only dereferencing a raw pointer would not be.
fn adjacent_share(ps: &[Box<Point>]) -> f64 {
    let addr = |p: &Point| std::ptr::from_ref(p) as usize;
    let step = size_of::<Point>();
    let adjacent = ps.windows(2).filter(|w| addr(&w[1]) == addr(&w[0]) + step);
    adjacent.count() as f64 / (ps.len() - 1) as f64
}

fn measure_pointers(n: usize, rng: &mut Rng) -> (Row, f64, f64) {
    // Shuffle: a structure built up over its lifetime is not laid out in
    // allocation order, and the prefetcher can no longer guess what is next.
    // Shuffling moves 8-byte pointers, never the points. It also means the
    // boxes are freed in scattered order.
    let (build, drop) = best_build_and_drop(|| build_pointers(n), |v| rng.shuffle(v));

    let (blocks0, bytes0) = heap();
    let mut pointers = build_pointers(n);
    let (blocks1, bytes1) = heap();

    // Boxes made in a tight loop sit next to each other, which is the BEST case.
    // Measure that first, then shuffle.
    let in_alloc_order = best_ns_per_op(|| {
        black_box(sum_pointers(black_box(&pointers)));
    });
    let adjacent = adjacent_share(&pointers);
    rng.shuffle(&mut pointers);
    let sum_ns = best_ns_per_op(|| {
        black_box(sum_pointers(black_box(&pointers)));
    });
    let row = Row {
        build,
        sum_ns,
        drop,
        blocks: blocks1 - blocks0,
        bytes: bytes1 - bytes0,
    };
    (row, in_alloc_order, adjacent)
}

fn main() {
    const N: usize = 1_000_000;
    let mut rng = Rng(3);

    let val = measure_values(N);
    let (ptr, ptr_in_order, adjacent) = measure_pointers(N, &mut rng);

    println!("summing {N} points, held two ways\n");

    // like Go's Duration printing: microseconds below a millisecond
    let ms = |d: Duration| match d.as_secs_f64() {
        s if s < 1e-3 => format!("{:.1}us", s * 1e6),
        s => format!("{:.1}ms", s * 1e3),
    };
    println!(
        "{:<24} {:>10} {:>14} {:>10} {:>18} {:>10}",
        "", "build", "sum (ns/op)", "drop", "live heap blocks", "heap MB"
    );
    for (name, r) in [
        ("Vec<Point> (contiguous)", &val),
        ("Vec<Box<Point>> (chased)", &ptr),
    ] {
        println!(
            "{name:<24} {:>10} {:>14.0} {:>10} {:>18} {:>10.1}",
            ms(r.build),
            r.sum_ns,
            ms(r.drop),
            r.blocks,
            r.bytes as f64 / (1 << 20) as f64
        );
    }
    println!(
        "{:<24} {:>10} {:>13.1}x {:>10} {:>17.0}x {:>9.1}x",
        "ratio",
        "-", // the Vec<Point> timings are one allocation and one free, so near
        // zero and dominated by page faults: their ratio would be noise
        ptr.sum_ns / val.sum_ns,
        "-",
        ptr.blocks as f64 / val.blocks as f64,
        ptr.bytes as f64 / val.bytes as f64
    );
    println!(
        "\nthe same boxes before the shuffle (allocation order): {ptr_in_order:.0} ns/op, {:.1}x the values,",
        ptr_in_order / val.sum_ns
    );
    println!("against {:.1}x after it.", ptr.sum_ns / val.sum_ns);

    println!();
    println!("the pointer version pays for its indirection three ways, none of them in the");
    println!("Big-O. They are not quite Go's three:");
    println!();
    println!(
        "  1. memory     {:.1}x the bytes asked of the allocator: an {}-byte pointer AND a",
        ptr.bytes as f64 / val.bytes as f64,
        size_of::<Box<Point>>()
    );
    println!(
        "                {}-byte pointee, against {} for the value alone",
        size_of::<Point>(),
        size_of::<Point>()
    );
    println!("  2. locality   each element is a separate block. Boxes made in a loop still sit");
    println!(
        "                side by side ({:.0}% of neighbours exactly {} bytes apart); the walk",
        100.0 * adjacent,
        size_of::<Point>()
    );
    println!("                turns into a chase once the Vec is no longer in allocation");
    println!("                order -- the example 8 effect");
    println!("  3. allocator  there is no collector, so nothing is traced, ever. What takes its");
    println!(
        "                place is {} allocator calls to build ({}) and as many",
        ptr.blocks,
        ms(ptr.build)
    );
    println!(
        "                frees to drop ({}), against one of each for Vec<Point>",
        ms(ptr.drop)
    );
    println!("                -- paid once, not on every cycle");
    println!();
    println!("both loops are Theta(n). One walks memory, the other chases it.");
    println!("In Rust too, choosing Vec<T> over Vec<Box<T>> is a decision about constants:");
    println!(
        "{:.1}x on the sum, {} fewer allocator calls, {:.1}x fewer bytes.",
        ptr.sum_ns / val.sum_ns,
        ptr.blocks - val.blocks,
        ptr.bytes as f64 / val.bytes as f64
    );
    println!();
    println!("when you DO need Box<T>: elements are huge and moved often, the type is");
    println!("recursive, or the type is polymorphic (Box<dyn Trait>). Sharing between owners");
    println!("is Rc/Arc in Rust, not a plain pointer.");
}
```

**Rust sample output:**

```
summing 1000000 points, held two ways

                              build    sum (ns/op)       drop   live heap blocks    heap MB
Vec<Point> (contiguous)     209.5us         162489      0.2us                  1       15.3
Vec<Box<Point>> (chased)     12.2ms         712323     52.1ms            1000001       22.9
ratio                             -           4.4x          -           1000001x       1.5x

the same boxes before the shuffle (allocation order): 268175 ns/op, 1.7x the values,
against 4.4x after it.

the pointer version pays for its indirection three ways, none of them in the
Big-O. They are not quite Go's three:

  1. memory     1.5x the bytes asked of the allocator: an 8-byte pointer AND a
                16-byte pointee, against 16 for the value alone
  2. locality   each element is a separate block. Boxes made in a loop still sit
                side by side (100% of neighbours exactly 16 bytes apart); the walk
                turns into a chase once the Vec is no longer in allocation
                order -- the example 8 effect
  3. allocator  there is no collector, so nothing is traced, ever. What takes its
                place is 1000001 allocator calls to build (12.2ms) and as many
                frees to drop (52.1ms), against one of each for Vec<Point>
                -- paid once, not on every cycle

both loops are Theta(n). One walks memory, the other chases it.
In Rust too, choosing Vec<T> over Vec<Box<T>> is a decision about constants:
4.4x on the sum, 1000000 fewer allocator calls, 1.5x fewer bytes.

when you DO need Box<T>: elements are huge and moved often, the type is
recursive, or the type is polymorphic (Box<dyn Trait>). Sharing between owners
is Rc/Arc in Rust, not a plain pointer.
```

> **Go vs Rust:** Go's third tax, GC tracing, does not exist in Rust, so that column became what Rust pays instead: one allocator call and one free per element (on a quiet machine build took 12-13 ms and drop 52-62 ms for the 1,000,001 blocks, against a fraction of a millisecond and a few microseconds at most for the single `Vec<Point>` block; drop reached 127 ms when the machine was busy). The drop is paid once, whereas Go's 3.7 ms is per GC cycle, so the two columns are not comparable. Live blocks are 1,000,001 against 1 (Go: 1,000,206 against 174, runtime objects included). Go's 'memory: 2x the bytes' is 1.5x for this 16-byte struct (8-byte pointer + 16-byte pointee against 16: 22.9 vs 15.3 MB here, the same arithmetic in Go); 2x is the `[]*int` case of example 6. The sum ratio is 4-5.3x on a quiet machine (3-14x when busy) against Go's 2.4x, because Rust's contiguous loop is faster (160-210 us against Go's 311 us; `--emit asm` shows NEON `ld2`/`add` vector instructions for it) while the pointer chase is the same in both (690-920 us against 758 us). Before the shuffle the boxes cost 1.5-2x on a quiet machine, with every neighbour exactly 16 bytes apart. clippy's `vec_box` lint flags this very type, hence the targeted `allow`.

**Complexity:** both sums are Θ(n) · the pointer version costs 2× memory, 2.4× time, 59× GC — and the shuffle is not artificial, it is what any structure built up over its lifetime looks like

---

## 10. Size classes: what you asked for vs what you got

`🟡 medium` · *Allocator rounding*

Go's allocator keeps free lists for a fixed set of **size classes** and rounds every allocation up to
the next one. Ask for 33 bytes, get billed 48. On a tree with millions of nodes this is real memory.

**Steps:**

1. Measure `AllocedBytesPerOp` for `make([]byte, k)` across k.
2. Read the size-class boundaries straight off the jumps.
3. Apply it to two node structs and scale to 10M nodes.

```go
package main

import (
	"fmt"
	"testing"
	"unsafe"
)

var sinkB []byte

// Go's allocator does not hand out arbitrary sizes. It keeps free lists for a
// fixed set of SIZE CLASSES (8, 16, 24, 32, 48, 64, 80, 96, 112, 128, ...) and
// rounds every allocation UP to the next one. The rounding is invisible from
// the language and very visible in your RSS.
func allocatedFor(size int) uint64 {
	res := testing.Benchmark(func(b *testing.B) {
		b.ReportAllocs()
		for b.Loop() {
			sinkB = make([]byte, size)
		}
	})
	return uint64(res.AllocedBytesPerOp())
}

type node24 struct { // 8 + 8 + 8
	key, val int
	next     *node24
}

type nodeFat struct { // 8 + 8 + 8 + 8 + 1 -> 40 declared, 48 allocated
	key, val int
	next     *nodeFat
	weight   float64
	deleted  bool
}

func main() {
	fmt.Println("what you ask for vs what you get:")
	fmt.Println()
	fmt.Printf("%10s %14s %10s %10s\n", "requested", "allocated", "wasted", "waste %")

	for _, size := range []int{1, 8, 9, 16, 17, 24, 25, 32, 33, 48, 49, 64, 65, 96, 128, 129} {
		got := allocatedFor(size)
		wasted := got - uint64(size)
		fmt.Printf("%10d %14d %10d %9.0f%%\n",
			size, got, wasted, 100*float64(wasted)/float64(got))
	}

	fmt.Println()
	fmt.Println("the jumps are the size classes. Ask for 33 bytes and you are billed 48.")
	fmt.Println()
	fmt.Println("(the first rows look free because objects under 16 bytes with no pointers")
	fmt.Println(" go through the TINY ALLOCATOR, which packs several of them into one block.")
	fmt.Println(" Add a pointer field and they rejoin the size-class table.)")

	fmt.Println()
	fmt.Println("why a data-structures course cares -- node sizes:")
	var a node24
	var b nodeFat
	fmt.Printf("  node24: Sizeof = %2d, allocated = %2d, waste %d bytes/node\n",
		unsafe.Sizeof(a), allocatedFor(int(unsafe.Sizeof(a))), allocatedFor(int(unsafe.Sizeof(a)))-uint64(unsafe.Sizeof(a)))
	fmt.Printf("  nodeFat: Sizeof = %2d, allocated = %2d, waste %d bytes/node\n",
		unsafe.Sizeof(b), allocatedFor(int(unsafe.Sizeof(b))), allocatedFor(int(unsafe.Sizeof(b)))-uint64(unsafe.Sizeof(b)))

	const n = 10_000_000
	waste := allocatedFor(int(unsafe.Sizeof(b))) - uint64(unsafe.Sizeof(b))
	fmt.Printf("\n  a %d-node tree of nodeFat wastes %.0f MB on rounding alone.\n",
		n, float64(waste)*n/(1<<20))

	fmt.Println()
	fmt.Println("two ways out, both of which this plan uses repeatedly:")
	fmt.Println("  1. shrink the struct to land on a class boundary (drop a field, use")
	fmt.Println("     int32 indices instead of 8-byte pointers -- example 16)")
	fmt.Println("  2. stop allocating per node: put the nodes in ONE []T and link them by")
	fmt.Println("     index. One allocation, no rounding, no GC pointers (examples 9, 13, 16)")
	fmt.Println()
	fmt.Println("note this is rounding on top of the struct padding from example 4:")
	fmt.Println("a badly ordered struct pads to 40, then the allocator rounds that to 48.")
}
```

**Output** (allocation sizes are deterministic — this table is stable):

```
what you ask for vs what you get:

 requested      allocated     wasted    waste %
         1              1          0         0%
         8              8          0         0%
         9             16          7        44%
        16             16          0         0%
        17             24          7        29%
        24             24          0         0%
        25             32          7        22%
        32             32          0         0%
        33             48         15        31%
        48             48          0         0%
        49             64         15        23%
        64             64          0         0%
        65             80         15        19%
        96             96          0         0%
       128            128          0         0%
       129            144         15        10%

the jumps are the size classes. Ask for 33 bytes and you are billed 48.

(the first rows look free because objects under 16 bytes with no pointers
 go through the TINY ALLOCATOR, which packs several of them into one block.
 Add a pointer field and they rejoin the size-class table.)

why a data-structures course cares -- node sizes:
  node24: Sizeof = 24, allocated = 24, waste 0 bytes/node
  nodeFat: Sizeof = 40, allocated = 48, waste 8 bytes/node

  a 10000000-node tree of nodeFat wastes 76 MB on rounding alone.

two ways out, both of which this plan uses repeatedly:
  1. shrink the struct to land on a class boundary (drop a field, use
     int32 indices instead of 8-byte pointers -- example 16)
  2. stop allocating per node: put the nodes in ONE []T and link them by
     index. One allocation, no rounding, no GC pointers (examples 9, 13, 16)

note this is rounding on top of the struct padding from example 4:
a badly ordered struct pads to 40, then the allocator rounds that to 48.
```

Note this **stacks on top of example 4**: a badly ordered struct pads to 40 bytes, and then the
allocator rounds that 40 up to 48.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::{BTreeSet, HashMap};
use std::hint::black_box;
use std::mem::size_of;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Go's testing.B reports AllocedBytesPerOp, the bytes the RUNTIME billed, after
// its own size-class rounding. Rust's allocator interface only ever sees the
// Layout the program asked for, so a counting allocator reports REQUESTED
// bytes; the rounding happens inside the system's malloc, out of its sight.
// A global allocator is an `unsafe` trait by definition; this one only counts
// and forwards every call to the system allocator.
struct Counting;
static REQUESTED: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        REQUESTED.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        REQUESTED.fetch_add(new, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

// std has no malloc_size / malloc_usable_size, and this program uses no crate
// and no FFI. What std CAN do is read addresses (safe) and subtract them.
// Allocate many blocks of one size, keep them all alive, sort the addresses:
// the most common gap between neighbours is what the allocator spends per
// block, rounding and any header included. That is the Rust twin of Go's
// "allocated" column. It is a measurement of this machine's allocator.
const BLOCKS: usize = 20_000;

fn pitch(mut addrs: Vec<usize>) -> usize {
    addrs.sort_unstable();
    let mut gaps: HashMap<usize, usize> = HashMap::new();
    for w in addrs.windows(2) {
        *gaps.entry(w[1] - w[0]).or_default() += 1;
    }
    let (gap, _) = gaps
        .into_iter()
        .max_by_key(|&(gap, count)| (count, std::cmp::Reverse(gap)))
        .expect("more than one block");
    gap
}

/// (bytes requested of the allocator, bytes the allocator spends) for a block of `size` bytes.
fn allocated_for(size: usize) -> (usize, usize) {
    let before = REQUESTED.load(Relaxed);
    let probe: Vec<u8> = Vec::with_capacity(size);
    let counted = REQUESTED.load(Relaxed) - before;
    drop(probe);

    let blocks: Vec<Vec<u8>> = (0..BLOCKS).map(|_| Vec::with_capacity(size)).collect();
    let addrs = blocks.iter().map(|b| b.as_ptr() as usize).collect();
    (counted, pitch(addrs))
}

/// What the allocator spends per `Box<T>`, for a real struct instead of raw bytes.
fn allocated_for_box<T>(make: impl Fn(i64) -> T) -> usize {
    let blocks: Vec<_> = (0..BLOCKS as i64).map(|i| Box::new(make(i))).collect();
    pitch(
        blocks
            .iter()
            .map(|b| std::ptr::from_ref(&**b) as usize)
            .collect(),
    )
}

struct Node24 {
    // 8 + 8 + 8; the fields exist so the size is real, only the size is measured
    #[allow(dead_code)]
    key: i64,
    #[allow(dead_code)]
    val: i64,
    #[allow(dead_code)]
    next: Option<Box<Node24>>,
}

struct NodeFat {
    // 8 + 8 + 8 + 8 + 1 = 33 bytes of fields
    #[allow(dead_code)]
    key: i64,
    #[allow(dead_code)]
    val: i64,
    #[allow(dead_code)]
    next: Option<Box<NodeFat>>,
    #[allow(dead_code)]
    weight: f64,
    #[allow(dead_code)]
    deleted: bool,
}

fn main() {
    println!("what you ask for vs what you get:\n");
    println!(
        "{:>10} {:>10} {:>12} {:>10} {:>10}",
        "requested", "counted", "allocated", "wasted", "waste %"
    );

    let mut pitches = BTreeSet::new();
    for size in [
        1usize, 8, 9, 16, 17, 24, 25, 32, 33, 48, 49, 64, 65, 96, 128, 129,
    ] {
        let (counted, got) = allocated_for(size);
        pitches.insert(got);
        let wasted = got - size;
        println!(
            "{size:>10} {counted:>10} {got:>12} {wasted:>10} {:>9.0}%",
            100.0 * wasted as f64 / got as f64
        );
    }

    println!();
    println!("'counted' is what a GlobalAlloc sees: the requested size, every row. The");
    println!("rounding is invisible to it -- and to the language -- but 'allocated' shows it.");
    let classes: Vec<String> = pitches.iter().map(|p| p.to_string()).collect();
    println!(
        "sizes this allocator actually spends: {}",
        classes.join(", ")
    );
    println!(
        "every one a multiple of 16: {}",
        pitches.iter().all(|p| p % 16 == 0)
    );
    println!();
    println!("(Go's allocator has a TINY ALLOCATOR that packs several pointer-free objects");
    println!(" under 16 bytes into one block, so its first rows are free. Nothing like it");
    println!(" shows up here: the 1-byte block costs the same as the 16-byte one.)");

    println!();
    println!("why a data-structures course cares -- node sizes:");
    let node24 = allocated_for_box(|i| Node24 {
        key: black_box(i),
        val: i,
        next: None,
    });
    let node_fat = allocated_for_box(|i| NodeFat {
        key: black_box(i),
        val: i,
        next: None,
        weight: 0.5,
        deleted: false,
    });
    let (s24, sfat) = (size_of::<Node24>(), size_of::<NodeFat>());
    println!(
        "  Node24:  size_of = {s24:>2}, allocated = {node24:>2}, waste {} bytes/node",
        node24 - s24
    );
    println!(
        "  NodeFat: size_of = {sfat:>2}, allocated = {node_fat:>2}, waste {} bytes/node",
        node_fat - sfat
    );

    const N: usize = 10_000_000;
    let waste = node_fat - sfat;
    println!(
        "\n  a {N}-node tree of NodeFat wastes {:.0} MB on rounding alone.",
        (waste * N) as f64 / (1 << 20) as f64
    );

    println!();
    println!("two ways out, both of which this plan uses repeatedly:");
    println!("  1. shrink the struct to land on a class boundary (drop a field, use");
    println!("     u32 indices instead of 8-byte pointers). The boundaries are the");
    println!("     allocator's, not Go's: here 24 is NOT free and 16 or 32 is");
    println!("  2. stop allocating per node: put the nodes in ONE Vec<T> and link them by");
    println!("     index. One allocation, no rounding, no per-node block");
    println!();
    // one block for all the nodes: the allocator rounds that block once, not each node
    let before = REQUESTED.load(Relaxed);
    let arena: Vec<NodeFat> = Vec::with_capacity(BLOCKS);
    let per_node = (REQUESTED.load(Relaxed) - before) / BLOCKS;
    black_box(&arena);
    println!("how much the second saves, per node: a Vec<NodeFat> asks for {per_node} bytes per");
    println!(
        "node in one block, so there is no per-node rounding; Box<NodeFat> spends {node_fat}."
    );
    println!();
    let fields = 3 * size_of::<i64>() + size_of::<f64>() + size_of::<bool>();
    println!("note this is rounding on top of the struct padding from example 4:");
    println!(
        "{fields} bytes of fields pad to {sfat}, then the allocator rounds that to {node_fat}."
    );
    println!("(Rust may reorder fields to shrink padding; here no order beats {sfat}.)");
}
```

**Rust sample output:**

```
what you ask for vs what you get:

 requested    counted    allocated     wasted    waste %
         1          1           16         15        94%
         8          8           16          8        50%
         9          9           16          7        44%
        16         16           16          0         0%
        17         17           32         15        47%
        24         24           32          8        25%
        25         25           32          7        22%
        32         32           32          0         0%
        33         33           48         15        31%
        48         48           48          0         0%
        49         49           64         15        23%
        64         64           64          0         0%
        65         65           80         15        19%
        96         96           96          0         0%
       128        128          128          0         0%
       129        129          144         15        10%

'counted' is what a GlobalAlloc sees: the requested size, every row. The
rounding is invisible to it -- and to the language -- but 'allocated' shows it.
sizes this allocator actually spends: 16, 32, 48, 64, 80, 96, 128, 144
every one a multiple of 16: true

(Go's allocator has a TINY ALLOCATOR that packs several pointer-free objects
 under 16 bytes into one block, so its first rows are free. Nothing like it
 shows up here: the 1-byte block costs the same as the 16-byte one.)

why a data-structures course cares -- node sizes:
  Node24:  size_of = 24, allocated = 32, waste 8 bytes/node
  NodeFat: size_of = 40, allocated = 48, waste 8 bytes/node

  a 10000000-node tree of NodeFat wastes 76 MB on rounding alone.

two ways out, both of which this plan uses repeatedly:
  1. shrink the struct to land on a class boundary (drop a field, use
     u32 indices instead of 8-byte pointers). The boundaries are the
     allocator's, not Go's: here 24 is NOT free and 16 or 32 is
  2. stop allocating per node: put the nodes in ONE Vec<T> and link them by
     index. One allocation, no rounding, no per-node block

how much the second saves, per node: a Vec<NodeFat> asks for 40 bytes per
node in one block, so there is no per-node rounding; Box<NodeFat> spends 48.

note this is rounding on top of the struct padding from example 4:
33 bytes of fields pad to 40, then the allocator rounds that to 48.
(Rust may reorder fields to shrink padding; here no order beats 40.)
```

> **Go vs Rust:** Go's size classes (8, 16, 24, 32, 48, ...) and its tiny allocator do not exist here: the system allocator on this machine (macOS) spends multiples of 16 bytes, so 1 byte costs 16 (Go: 1), 24 costs 32 (Go: 24) and 33 costs 48 (Go: 48). `Node24` therefore wastes 8 bytes per node where Go's wastes 0, while `NodeFat` wastes 8 in both (40 padded, 48 allocated, 76 MB at 10M nodes in both). Rust's `GlobalAlloc` only sees the requested layout (the 'counted' column equals 'requested' on every row), and std has no `malloc_size`/`malloc_usable_size` (no libc was used), so 'allocated' is measured as the most common address gap between 20,000 live blocks of one size: what the allocator spends per block, any header included. Another allocator (glibc, say) would print a different column; the table is deterministic here but machine-specific.

**Complexity:** Θ(1) per allocation · but it multiplies every per-node structure by up to 1.5×, and it is the second half of the argument for storing nodes in one slice

---

## 11. Interface boxing vs generics

`🟡 medium` · *Why containers take type parameters*

Before generics, a reusable container held `any`. That has a memory model: an interface value is two
words, and storing a non-pointer in one usually means heap-allocating a copy for it to point at.
Generics compile per concrete type and cost **nothing**.

**Steps:**

1. Hold a million ints as `[]int` and as `[]any`; compare memory.
2. Sum them three ways: concrete, generic, and through the interface.
3. Then shuffle the `[]any` — because a container filled over its lifetime is not contiguous.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"testing"
)

var sink int

// Before generics, a reusable container held `any`. That has a memory model:
// an interface value is two words (type pointer + data pointer), and storing a
// non-pointer value in one usually means allocating a copy on the heap for the
// data word to point at.

func sumInts(xs []int) int {
	total := 0
	for _, v := range xs {
		total += v
	}
	return total
}

func sumAny(xs []any) int {
	total := 0
	for _, v := range xs {
		total += v.(int) // a type assertion on every element
	}
	return total
}

type Number interface{ ~int | ~int64 | ~float64 }

// Generic code is COMPILED for the concrete type. No boxing, no assertions --
// this compiles to the same machine code as sumInts.
func sumGeneric[T Number](xs []T) T {
	var total T
	for _, v := range xs {
		total += v
	}
	return total
}

func heapMB() float64 {
	runtime.GC()
	var ms runtime.MemStats
	runtime.ReadMemStats(&ms)
	return float64(ms.HeapAlloc) / (1 << 20)
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
	const n = 1_000_000

	base := heapMB()
	ints := make([]int, n)
	for i := range ints {
		ints[i] = i + 1000 // above 255, so nothing comes from the small-int cache
	}
	afterInts := heapMB()

	boxed := make([]any, n)
	for i := range boxed {
		boxed[i] = i + 1000
	}
	afterBoxed := heapMB()

	// The same values, in a shuffled order. Allocating them back-to-back (above)
	// leaves them contiguous, which is the BEST case a []any ever gets. A real
	// container filled over its lifetime looks like this one instead.
	scattered := make([]any, n)
	copy(scattered, boxed)
	rng := rand.New(rand.NewPCG(3, 5))
	rng.Shuffle(n, func(i, j int) { scattered[i], scattered[j] = scattered[j], scattered[i] })

	fmt.Printf("holding %d ints:\n\n", n)
	fmt.Printf("  []int    %7.1f MB   (%.0f bytes/element)\n",
		afterInts-base, (afterInts-base)*(1<<20)/n)
	fmt.Printf("  []any    %7.1f MB   (%.0f bytes/element: 16-byte header + a boxed int)\n",
		afterBoxed-afterInts, (afterBoxed-afterInts)*(1<<20)/n)
	fmt.Printf("  ratio    %7.1fx\n", (afterBoxed-afterInts)/(afterInts-base))

	tInts := nsPerOp(func() { sink += sumInts(ints) })
	tGen := nsPerOp(func() { sink += sumGeneric(ints) })
	tAny := nsPerOp(func() { sink += sumAny(boxed) })
	tScattered := nsPerOp(func() { sink += sumAny(scattered) })

	fmt.Println()
	fmt.Printf("%-30s %14s %14s\n", "summing them", "ns/op", "vs []int")
	fmt.Printf("%-30s %14.0f %14s\n", "sumInts([]int)", tInts, "1.0x")
	fmt.Printf("%-30s %14.0f %13.1fx\n", "sumGeneric([]int)", tGen, tGen/tInts)
	fmt.Printf("%-30s %14.0f %13.1fx\n", "sumAny([]any) contiguous", tAny, tAny/tInts)
	fmt.Printf("%-30s %14.0f %13.1fx\n", "sumAny([]any) scattered", tScattered, tScattered/tInts)

	runtime.KeepAlive(ints)
	runtime.KeepAlive(boxed)
	runtime.KeepAlive(scattered)

	fmt.Println()
	fmt.Println("generics cost nothing here: Go compiles sumGeneric for int and produces")
	fmt.Println("the same loop as the hand-written version.")
	fmt.Println()
	fmt.Println("note the two []any rows. Filled back-to-back the boxes land contiguously")
	fmt.Println("and it costs only ~1.2x -- that is the BEST case, and it is what a naive")
	fmt.Println("benchmark measures. Shuffle them, as any real container ends up, and the")
	fmt.Println("pointer chase from example 8 doubles it again.")
	fmt.Println()
	fmt.Println("[]any costs on every axis at once:")
	fmt.Println("  memory   16 bytes of interface header per element, PLUS a separate")
	fmt.Println("           heap object per value (example 5's boxing)")
	fmt.Println("  locality the values are scattered, so the sum is a pointer chase (example 8)")
	fmt.Println("  CPU      a type assertion per element")
	fmt.Println("  GC       a million more pointers to trace (example 9)")
	fmt.Println()
	fmt.Println("the rule for this plan: containers take TYPE PARAMETERS, not `any`.")
	fmt.Println("Stack[T], Queue[T], Tree[T] -- never Stack of interface{}.")
	fmt.Println()
	fmt.Println("(this is also why container/list and container/heap, which predate")
	fmt.Println(" generics and store `any`, lose to a hand-rolled generic equivalent.)")
}
```

**Sample output:**

```
holding 1000000 ints:

  []int        7.6 MB   (8 bytes/element)
  []any       22.9 MB   (24 bytes/element: 16-byte header + a boxed int)
  ratio        3.0x

summing them                            ns/op       vs []int
sumInts([]int)                         264887           1.0x
sumGeneric([]int)                      253573           1.0x
sumAny([]any) contiguous               316801           1.2x
sumAny([]any) scattered                670359           2.5x

generics cost nothing here: Go compiles sumGeneric for int and produces
the same loop as the hand-written version.

note the two []any rows. Filled back-to-back the boxes land contiguously
and it costs only ~1.2x -- that is the BEST case, and it is what a naive
benchmark measures. Shuffle them, as any real container ends up, and the
pointer chase from example 8 doubles it again.

[]any costs on every axis at once:
  memory   16 bytes of interface header per element, PLUS a separate
           heap object per value (example 5's boxing)
  locality the values are scattered, so the sum is a pointer chase (example 8)
  CPU      a type assertion per element
  GC       a million more pointers to trace (example 9)

the rule for this plan: containers take TYPE PARAMETERS, not `any`.
Stack[T], Queue[T], Tree[T] -- never Stack of interface{}.

(this is also why container/list and container/heap, which predate
 generics and store `any`, lose to a hand-rolled generic equivalent.)
```

Two readings matter here. **Generics are free** — 1.0× against the hand-written loop, because Go
compiles `sumGeneric` for `int`. And the naive `[]any` benchmark is *too kind*: filled back-to-back
the boxes land contiguously and it costs only 1.2×. Shuffled, as any real container ends up, example
8's penalty doubles it again.

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::any::Any;
use std::collections::HashMap;
use std::hint::black_box;
use std::mem::size_of;
use std::ops::Add;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// Go reads runtime.MemStats.HeapAlloc after a forced GC. Rust has no GC, so a
// counting allocator keeps the exact bytes (and allocator calls) instead.
// A global allocator is an `unsafe` trait by definition; this one only does
// arithmetic and forwards every call to the system allocator.
struct Counting;
static CALLS: AtomicUsize = AtomicUsize::new(0);
static LIVE: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        CALLS.fetch_add(1, Relaxed);
        LIVE.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.dealloc(p, l) }
    }
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        CALLS.fetch_add(1, Relaxed);
        LIVE.fetch_add(new, Relaxed);
        LIVE.fetch_sub(l.size(), Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// (allocator calls, live bytes added) while running `f`, and what it built.
fn measure<R>(f: impl FnOnce() -> R) -> (usize, usize, R) {
    let (c0, l0) = (CALLS.load(Relaxed), LIVE.load(Relaxed));
    let r = f();
    (CALLS.load(Relaxed) - c0, LIVE.load(Relaxed) - l0, r)
}

// Go's testing.Benchmark / b.Loop becomes this tiny harness: run the closure
// until 200 ms have elapsed and report ns per call. The machine is shared and
// timings are noisy, so each figure is the best of 3.
fn ns_per_op(mut f: impl FnMut()) -> f64 {
    let once = |f: &mut dyn FnMut()| {
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
    };
    (0..3).map(|_| once(&mut f)).fold(f64::INFINITY, f64::min)
}

// Go's math/rand: seeded splitmix64 and a Fisher-Yates shuffle.
struct Rng(u64);

impl Rng {
    fn next_u64(&mut self) -> u64 {
        self.0 = self.0.wrapping_add(0x9e3779b97f4a7c15);
        let mut z = self.0;
        z = (z ^ (z >> 30)).wrapping_mul(0xbf58476d1ce4e5b9);
        z = (z ^ (z >> 27)).wrapping_mul(0x94d049bb133111eb);
        z ^ (z >> 31)
    }
    fn below(&mut self, n: u64) -> u64 {
        ((self.next_u64() as u128 * n as u128) >> 64) as u64
    }
    fn shuffle<T>(&mut self, xs: &mut [T]) {
        for i in (1..xs.len()).rev() {
            xs.swap(i, self.below(i as u64 + 1) as usize);
        }
    }
}

// Go's `any` is two words (type pointer + data pointer), and storing a
// non-pointer in one usually allocates a copy for the data word to point at.
// Rust's two spellings of that idea are trait objects: `Box<dyn Any>` is the
// literal twin of `any` (you ask for the type back with downcast_ref, as Go
// does with v.(int)), and `Box<dyn Value>` is how Rust code writes real
// polymorphism: a method call through a vtable. Both are a 16-byte fat pointer
// (data pointer + vtable pointer) to a separately allocated i64.
trait Value {
    fn get(&self) -> i64;
}
impl Value for i64 {
    fn get(&self) -> i64 {
        *self
    }
}

fn sum_ints(xs: &[i64]) -> i64 {
    let mut total = 0;
    for &v in xs {
        total += v;
    }
    total
}

fn sum_any(xs: &[Box<dyn Any>]) -> i64 {
    xs.iter()
        .map(|v| *v.downcast_ref::<i64>().expect("an i64")) // a type check per element
        .sum()
}

fn sum_dyn(xs: &[Box<dyn Value>]) -> i64 {
    xs.iter().map(|v| v.get()).sum() // a virtual call per element
}

// Go's `Number interface{ ~int | ~int64 | ~float64 }`: a trait bound.
trait Number: Copy + Default + Add<Output = Self> {}
impl<T: Copy + Default + Add<Output = T>> Number for T {}

// Generic code is MONOMORPHISED: rustc compiles a separate copy for each
// concrete type it is used with. No boxing, no type checks, no vtable.
fn sum_generic<T: Number>(xs: &[T]) -> T {
    let mut total = T::default();
    for &v in xs {
        total = total + v;
    }
    total
}

/// The most common gap between neighbouring blocks: what the allocator spends
/// per block (std has no malloc_size; subtracting addresses is safe).
fn pitch(mut addrs: Vec<usize>) -> usize {
    addrs.sort_unstable();
    let mut gaps: HashMap<usize, usize> = HashMap::new();
    for w in addrs.windows(2) {
        *gaps.entry(w[1] - w[0]).or_default() += 1;
    }
    let (gap, _) = gaps
        .into_iter()
        .max_by_key(|&(gap, count)| (count, std::cmp::Reverse(gap)))
        .expect("more than one block");
    gap
}

fn main() {
    const N: usize = 1_000_000;
    let mb = |bytes: usize| bytes as f64 / (1 << 20) as f64;

    // No small-int cache or static copy in Rust, so the `+ 1000` Go needs to
    // dodge one changes nothing here; it is kept so the values are the same.
    let (ints_calls, ints_bytes, ints) =
        measure(|| (0..N as i64).map(|i| i + 1000).collect::<Vec<_>>());
    let (any_calls, any_bytes, mut boxed_any) = measure(|| {
        (0..N as i64)
            .map(|i| Box::new(i + 1000) as Box<dyn Any>)
            .collect::<Vec<_>>()
    });
    let (_, dyn_bytes, mut boxed_dyn) = measure(|| {
        (0..N as i64)
            .map(|i| Box::new(i + 1000) as Box<dyn Value>)
            .collect::<Vec<_>>()
    });

    println!("holding {N} ints:\n");
    println!(
        "  Vec<i64>              {:>5.1} MB   ({:.0} bytes/element, {ints_calls} allocator call)",
        mb(ints_bytes),
        ints_bytes as f64 / N as f64
    );
    println!(
        "  Vec<Box<dyn Any>>     {:>5.1} MB   ({:.0} bytes/element: {}-byte fat pointer + a boxed i64, {any_calls} calls)",
        mb(any_bytes),
        any_bytes as f64 / N as f64,
        size_of::<Box<dyn Any>>()
    );
    println!(
        "  Vec<Box<dyn Value>>   {:>5.1} MB   (the same: {}-byte fat pointer + a boxed i64)",
        mb(dyn_bytes),
        size_of::<Box<dyn Value>>()
    );
    println!(
        "  ratio                 {:>5.1}x",
        any_bytes as f64 / ints_bytes as f64
    );
    // Those are REQUESTED bytes. What the allocator spends on each 8-byte box,
    // and how many neighbours in the Vec sit exactly that far apart:
    let addr = |v: &dyn Any| std::ptr::from_ref(v).cast::<u8>() as usize;
    let box_pitch = pitch(boxed_any.iter().map(|b| addr(&**b)).collect());
    let adjacent = boxed_any
        .windows(2)
        .filter(|w| addr(&*w[1]) == addr(&*w[0]) + box_pitch)
        .count() as f64
        / (N - 1) as f64;
    println!(
        "\n  requested bytes only: the allocator spends {box_pitch} on each 8-byte box (example 10),\n  so the true cost is {} bytes/element, {:.1}x Vec<i64>",
        size_of::<Box<dyn Any>>() + box_pitch,
        (size_of::<Box<dyn Any>>() + box_pitch) as f64 / size_of::<i64>() as f64
    );

    let t_ints = ns_per_op(|| {
        black_box(sum_ints(black_box(&ints)));
    });
    let t_gen = ns_per_op(|| {
        black_box(sum_generic(black_box(&ints)));
    });
    let t_any = ns_per_op(|| {
        black_box(sum_any(black_box(&boxed_any)));
    });
    let t_dyn = ns_per_op(|| {
        black_box(sum_dyn(black_box(&boxed_dyn)));
    });

    // The same values, in a shuffled order. Allocating them back-to-back (above)
    // leaves them contiguous, which is the BEST case a boxed Vec ever gets. A
    // real container filled over its lifetime looks like this one instead.
    // Box is not Copy, so there is no second copy of the Vec to shuffle: the same
    // Vec is shuffled in place (it moves 16-byte pointers, never the i64s).
    let mut rng = Rng(3);
    rng.shuffle(&mut boxed_any);
    rng.shuffle(&mut boxed_dyn);
    let t_any_s = ns_per_op(|| {
        black_box(sum_any(black_box(&boxed_any)));
    });
    let t_dyn_s = ns_per_op(|| {
        black_box(sum_dyn(black_box(&boxed_dyn)));
    });

    println!();
    println!(
        "{:<40} {:>14} {:>14}",
        "summing them", "ns/op", "vs Vec<i64>"
    );
    println!("{:<40} {t_ints:>14.0} {:>14}", "sum_ints(&[i64])", "1.0x");
    for (name, t) in [
        ("sum_generic(&[i64])", t_gen),
        ("sum_any(&[Box<dyn Any>]) contiguous", t_any),
        ("sum_any(&[Box<dyn Any>]) scattered", t_any_s),
        ("sum_dyn(&[Box<dyn Value>]) contiguous", t_dyn),
        ("sum_dyn(&[Box<dyn Value>]) scattered", t_dyn_s),
    ] {
        println!("{name:<40} {t:>14.0} {:>13.1}x", t / t_ints);
    }

    // Keep the data alive until the last timing, like runtime.KeepAlive.
    black_box((&ints, &boxed_any, &boxed_dyn));

    println!();
    println!("generics cost nothing here: rustc compiles sum_generic::<i64> into the same");
    println!(
        "loop as the hand-written one ({:.1}x). Checked, not assumed: for these two functions",
        t_gen / t_ints
    );
    println!("`rustc --emit asm` gives an identical instruction sequence.");
    println!();
    println!(
        "note the contiguous and scattered rows. Filled back-to-back the boxes land side by side"
    );
    println!(
        "({:.0}% of neighbours exactly {box_pitch} bytes apart):",
        100.0 * adjacent
    );
    println!(
        "{:.1}x / {:.1}x for sum_any / sum_dyn -- that is the BEST case, and it is what a naive",
        t_any / t_ints,
        t_dyn / t_ints
    );
    println!(
        "benchmark measures. Shuffled, as any real container ends up, it is {:.1}x / {:.1}x:",
        t_any_s / t_ints,
        t_dyn_s / t_ints
    );
    println!(
        "{:.1}x / {:.1}x worse than contiguous, the pointer chase from example 8.",
        t_any_s / t_any,
        t_dyn_s / t_dyn
    );
    println!();
    println!("boxed trait objects cost on every axis at once:");
    println!("  memory   a 16-byte fat pointer per element, PLUS a separate heap block per");
    println!("           value ({any_calls} allocator calls against {ints_calls})");
    println!("  locality the values are scattered, so the sum is a pointer chase (example 8)");
    println!("  CPU      a virtual call per element: sum_dyn calls get() through the vtable;");
    println!("           sum_any calls type_id() through it and compares the 128-bit result");
    println!("  free     dropping them is a million frees (example 9)");
    println!();
    println!("the rule for this plan: containers take TYPE PARAMETERS, not trait objects.");
    println!("Stack<T>, Queue<T>, Tree<T> -- never Stack of Box<dyn Any>.");
    println!();
    println!("(Rust's std containers -- Vec<T>, VecDeque<T>, BinaryHeap<T> -- were generic");
    println!(" from 1.0, so there is no pre-generics container to lose to a rewrite.)");
}
```

**Rust sample output:**

```
holding 1000000 ints:

  Vec<i64>                7.6 MB   (8 bytes/element, 1 allocator call)
  Vec<Box<dyn Any>>      22.9 MB   (24 bytes/element: 16-byte fat pointer + a boxed i64, 1000001 calls)
  Vec<Box<dyn Value>>    22.9 MB   (the same: 16-byte fat pointer + a boxed i64)
  ratio                   3.0x

  requested bytes only: the allocator spends 16 on each 8-byte box (example 10),
  so the true cost is 32 bytes/element, 4.0x Vec<i64>

summing them                                      ns/op    vs Vec<i64>
sum_ints(&[i64])                                  73808           1.0x
sum_generic(&[i64])                               73585           1.0x
sum_any(&[Box<dyn Any>]) contiguous              779023          10.6x
sum_any(&[Box<dyn Any>]) scattered              1235430          16.7x
sum_dyn(&[Box<dyn Value>]) contiguous            772968          10.5x
sum_dyn(&[Box<dyn Value>]) scattered            1228210          16.6x

generics cost nothing here: rustc compiles sum_generic::<i64> into the same
loop as the hand-written one (1.0x). Checked, not assumed: for these two functions
`rustc --emit asm` gives an identical instruction sequence.

note the contiguous and scattered rows. Filled back-to-back the boxes land side by side
(100% of neighbours exactly 16 bytes apart):
10.6x / 10.5x for sum_any / sum_dyn -- that is the BEST case, and it is what a naive
benchmark measures. Shuffled, as any real container ends up, it is 16.7x / 16.6x:
1.6x / 1.6x worse than contiguous, the pointer chase from example 8.

boxed trait objects cost on every axis at once:
  memory   a 16-byte fat pointer per element, PLUS a separate heap block per
           value (1000001 allocator calls against 1)
  locality the values are scattered, so the sum is a pointer chase (example 8)
  CPU      a virtual call per element: sum_dyn calls get() through the vtable;
           sum_any calls type_id() through it and compares the 128-bit result
  free     dropping them is a million frees (example 9)

the rule for this plan: containers take TYPE PARAMETERS, not trait objects.
Stack<T>, Queue<T>, Tree<T> -- never Stack of Box<dyn Any>.

(Rust's std containers -- Vec<T>, VecDeque<T>, BinaryHeap<T> -- were generic
 from 1.0, so there is no pre-generics container to lose to a rewrite.)
```

> **Go vs Rust:** Go's `any` becomes two Rust things: `Box<dyn Any>` (the literal twin: `downcast_ref::<i64>()` is the `v.(int)`) and `Box<dyn Value>` (a trait method, how Rust writes real polymorphism). Generics are monomorphised: `sum_generic::<i64>` and the hand-written loop compile to an identical instruction sequence (`--emit asm`), 0.9-1.1x here. Requested memory matches Go exactly (7.6 vs 22.9 MB, 24 bytes per element, 3.0x), but the system allocator rounds each 8-byte box up to 16, so the real cost is 32 bytes per element, 4.0x. The Go claim that contiguous boxes cost 'only ~1.2x' does not carry over: 7-12x over ten runs for both boxed rows, because the `Vec<i64>` loop is vectorised (72-126 us; Go's `sumInts`: 265 us) while each boxed element costs one indirect call through the vtable (`blr` for `type_id()` in sum_any's assembly), about 0.8 ns apiece, against Go's inline type compare (317 us for the whole loop). Shuffling made the boxed sums 1.4-2.9x slower again (Go: 2.1x); that row is the noisiest, 13-42x of the baseline, because other jobs share the machine's memory. `Box` is not `Copy`, so the scattered rows shuffle the same `Vec` in place instead of copying it first.

**Complexity:** all three sums are Θ(n) · `[]any` costs 3× the memory, a type assertion per element, and a million extra pointers for the GC — which is why every container in this plan is `Stack[T]`, never `Stack` of `any`

---

> Next tier: [🔴 hard](3-hard.md) — where layout starts beating asymptotics.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
