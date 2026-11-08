# Step 01 — Introduction · 🔴 Hard

Examples **11–14**. Each is a complete `package main` program: read the concept and steps,
then **retype the code block** into a scratch folder and run it.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
# type the example into main.go, then:
go run .
```

This tier is about the ways a measurement can be **wrong**, and the one cheap way to be sure your
code is **right**. Examples 11–13 print timings that vary by machine; example 14 is deterministic.

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

## 11. Big-O lies at small n

`🔴 hard` · *Crossover / why hybrid sorts exist*

Insertion sort is O(n²) and `slices.Sort` is O(n log n), so the textbook says this is no contest. The
textbook is talking about the limit. At n=16 the "inferior" algorithm is **20% faster**, because it
has no pivot selection, no recursion, no bookkeeping — just a tight loop over data that fits in L1.
This is not a curiosity: it's why Go's own `slices.Sort` (pdqsort) drops to insertion sort at
`length <= 12`.

**Steps:**

1. Write insertion sort; sort a *copy* so each iteration starts from the same unsorted data.
2. Put the `copy` **inside** the timed loop for both contenders, so they pay the same overhead.
3. Sweep n from 4 to 8192 and find where the winner flips.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// insertionSort is O(n^2) -- textbook-inferior to any O(n log n) sort.
func insertionSort(xs []int) {
	for i := 1; i < len(xs); i++ {
		v := xs[i]
		j := i - 1
		for j >= 0 && xs[j] > v {
			xs[j+1] = xs[j]
			j--
		}
		xs[j+1] = v
	}
}

func main() {
	rng := rand.New(rand.NewPCG(7, 11))

	fmt.Printf("%7s %18s %18s   %s\n", "n", "insertion ns/op", "slices.Sort ns/op", "winner")
	crossover := 0

	for _, n := range []int{4, 8, 16, 32, 64, 128, 1024, 8192} {
		src := make([]int, n)
		for i := range src {
			src[i] = rng.IntN(1 << 20)
		}

		// Copy fresh data inside the timed loop -- both contenders pay the same copy.
		insRes := testing.Benchmark(func(b *testing.B) {
			buf := make([]int, n)
			for b.Loop() {
				copy(buf, src)
				insertionSort(buf)
			}
		})
		stdRes := testing.Benchmark(func(b *testing.B) {
			buf := make([]int, n)
			for b.Loop() {
				copy(buf, src)
				slices.Sort(buf)
			}
		})

		winner := "insertion"
		if stdRes.NsPerOp() < insRes.NsPerOp() {
			winner = "slices.Sort"
			if crossover == 0 {
				crossover = n
			}
		}
		fmt.Printf("%7d %18d %18d   %s\n", n, insRes.NsPerOp(), stdRes.NsPerOp(), winner)
	}

	fmt.Println()
	fmt.Printf("the O(n^2) sort wins until n = %d, then loses by orders of magnitude\n", crossover)
	fmt.Println("this is not a paradox: Big-O describes the limit, not your actual input size")
	fmt.Println("Go's own slices.Sort (pdqsort) drops to insertion sort at n <= 12 for exactly this reason")
}
```

**Sample output** (timings vary; the crossover is typically 16–64):

```
      n    insertion ns/op  slices.Sort ns/op   winner
      4                  3                  4   insertion
      8                 11                 17   insertion
     16                 33                 40   insertion
     32                106                 97   slices.Sort
     64                418                256   slices.Sort
    128               1697                627   slices.Sort
   1024              97376               8418   slices.Sort
   8192            5953672              92974   slices.Sort

the O(n^2) sort wins until n = 32, then loses by orders of magnitude
this is not a paradox: Big-O describes the limit, not your actual input size
Go's own slices.Sort (pdqsort) drops to insertion sort at n <= 12 for exactly this reason
```

Watch the last two rows: 8× the input costs insertion sort **61×** the time (≈8²) and `slices.Sort`
**11×** (≈8·log). The asymptotics were right all along — they just weren't relevant at n=16.

**Rust:**

```rust
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

// The machine is shared and timings jitter, so take the fastest of three
// calibrated runs: noise only ever adds time, never removes it.
fn best_ns_per_op(mut f: impl FnMut()) -> f64 {
    (0..3)
        .map(|_| ns_per_op(&mut f))
        .fold(f64::INFINITY, f64::min)
}

// Rng: splitmix64, standing in for math/rand (std has no RNG). Seeded, so the
// input data is the same on every run; only the timings vary.
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

// insertion_sort is O(n^2) -- textbook-inferior to any O(n log n) sort.
// (Rust's own slice::sort_unstable falls back to insertion sort for
// len <= 20: MAX_LEN_ALWAYS_INSERTION_SORT in core/src/slice/sort/unstable/mod.rs,
// rustc 1.98.1. Go's slices.Sort does so at len <= 12.)
// Go walks j down to -1; a usize index cannot, so j here is "one past" the
// slot being compared (xs[j - 1] is the element to the left).
//
// #[inline(never)] keeps this function's machine code independent of the
// benchmark loop around it: with default inlining, the same source measured
// ~50% slower at n = 32..128 in one build than in another (cause not
// investigated), which would make the table below unreproducible.
#[inline(never)]
fn insertion_sort(xs: &mut [i64]) {
    for i in 1..xs.len() {
        let v = xs[i];
        let mut j = i;
        while j > 0 && xs[j - 1] > v {
            xs[j] = xs[j - 1];
            j -= 1;
        }
        xs[j] = v;
    }
}

fn main() {
    let mut rng = Rng::new(7);

    println!(
        "{:>7} {:>18} {:>20} {:>12}   winner",
        "n", "insertion ns/op", "sort_unstable ns/op", "insertion/std"
    );
    let mut rows = Vec::new();

    for n in [4usize, 8, 16, 32, 64, 128, 1024, 8192] {
        let src: Vec<i64> = (0..n).map(|_| rng.below(1 << 20) as i64).collect();
        let mut buf = vec![0i64; n];

        // Correctness before speed: both sorts must produce the same order.
        let mut by_insertion = src.clone();
        insertion_sort(&mut by_insertion);
        let mut by_std = src.clone();
        by_std.sort_unstable();
        assert_eq!(by_insertion, by_std);

        // Copy fresh data inside the timed loop -- both contenders pay the same copy.
        let ins_ns = best_ns_per_op(|| {
            buf.copy_from_slice(black_box(&src));
            insertion_sort(black_box(&mut buf));
        });
        let std_ns = best_ns_per_op(|| {
            buf.copy_from_slice(black_box(&src));
            black_box(&mut buf).sort_unstable();
        });

        let winner = if std_ns < ins_ns {
            "sort_unstable"
        } else {
            "insertion"
        };
        println!(
            "{n:>7} {ins_ns:>18.1} {std_ns:>20.1} {:>11.2}x   {winner}",
            ins_ns / std_ns
        );
        rows.push((n, ins_ns, std_ns));
    }

    // Go finds a crossover at 16-64, where its pdqsort starts beating insertion
    // sort. Rust's sort_unstable has no such regime: for len <= 20 the standard
    // library *is* insertion sort (see the comment on insertion_sort). So
    // report the ratio at each regime instead of a crossover that would jump
    // between runs.
    let ratio_at = |n: usize| {
        let &(_, ins, std) = rows.iter().find(|r| r.0 == n).unwrap();
        ins / std
    };
    let small: Vec<f64> = [4, 8, 16].iter().map(|&n| ratio_at(n)).collect();
    let lo = small.iter().copied().fold(f64::INFINITY, f64::min);
    let hi = small.iter().copied().fold(0.0, f64::max);
    println!();
    println!(
        "n <= 16:   insertion/std spans {lo:.2}x..{hi:.2}x   (sort_unstable is insertion sort here too)"
    );
    println!(
        "n = 32:    insertion/std is {:.2}x        (sort_unstable has switched algorithm)",
        ratio_at(32)
    );
    println!("n = 8192:  insertion/std is {:.0}x", ratio_at(8192));

    // Do the asymptotics show up? Compare the last two rows: 8x the input.
    let [.., (n0, ins0, std0), (n1, ins1, std1)] = rows[..] else {
        unreachable!("the sweep has more than two rows")
    };
    let grow = n1 as f64 / n0 as f64;
    let nlogn = grow * (n1 as f64).log2() / (n0 as f64).log2();
    println!(
        "n = {n0} -> {n1} ({grow:.0}x the input): insertion cost x{:.1} (n^2 predicts x{:.0}), sort_unstable x{:.1} (n log n predicts x{nlogn:.1})",
        ins1 / ins0,
        grow * grow,
        std1 / std0
    );
}
```

**Rust sample output:**

```
      n    insertion ns/op  sort_unstable ns/op insertion/std   winner
      4                4.8                  4.2        1.14x   sort_unstable
      8               11.5                  9.1        1.26x   sort_unstable
     16               37.6                 35.2        1.07x   sort_unstable
     32              113.1                 84.5        1.34x   sort_unstable
     64              332.8                159.0        2.09x   sort_unstable
    128             1590.4                387.0        4.11x   sort_unstable
   1024            77185.6               4476.3       17.24x   sort_unstable
   8192          4675875.0              50179.7       93.18x   sort_unstable

n <= 16:   insertion/std spans 1.07x..1.26x   (sort_unstable is insertion sort here too)
n = 32:    insertion/std is 1.34x        (sort_unstable has switched algorithm)
n = 8192:  insertion/std is 93x
n = 1024 -> 8192 (8x the input): insertion cost x60.6 (n^2 predicts x64), sort_unstable x11.2 (n log n predicts x10.4)
```

> **Go vs Rust:** Rust does not reproduce Go's small-n win for insertion sort: over six runs (best of three calibrated runs per cell) `sort_unstable` was ahead in every cell but one (n = 4, 0.99x in one run), and at n <= 16 the two stayed within 0.99x–1.32x of each other. The reason is in std's source, not in this output: `sort_unstable` is itself insertion sort for len <= 20 (`MAX_LEN_ALWAYS_INSERTION_SORT`, rustc 1.98.1; Go's threshold is 12), so at small n both contenders run the same algorithm and the gap is implementation detail (plausible, not measured: my version bounds-checks every index). The asymptotics do show up: insertion/std was 1.3–1.4x at n = 32 and 91–98x at n = 8192 (Go sample: 64x), and 8x the input cost insertion sort 54–61x (n^2 predicts 64x) and `sort_unstable` 10–11x (n log n predicts 10.4x).

**Complexity:** insertion O(n²) worst, O(n) on nearly-sorted input, in-place · pdqsort O(n log n) worst, O(log n) stack — both in full at [12](../../12-elementary-sorts.md) and [13](../../13-efficient-sorts.md)

---

## 12. The benchmark that measured nothing

`🔴 hard` · *Dead-code elimination / harness overhead*

The most important example in this lesson, and the least comfortable. Three standard ways to
benchmark a one-multiply function, and **all three report a number that isn't the function's cost**.
The fix isn't a trick — it's a control.

Two independent failures are on display:

1. **Dead-code elimination.** `square` is small and pure, so if its result is unused the compiler
   deletes the call. The famous "accumulate into a package-level sink" defense **does not save you
   here**: the compiler can still see through the whole loop.
2. **Harness overhead.** `b.Loop` prevents the elimination — but costs ~1.6 ns per iteration itself.
   A benchmark reporting 1.6 ns/op for a single multiply measured the loop, not the multiply. You
   only discover this by running an **empty** `b.Loop` as a control.

And a warning against over-correcting: DCE is *selective*. `fib(20)` discarded in the very same kind
of loop is **not** removed — it's recursive, can't be inlined, and can't be proven pure.

**Steps:**

1. Benchmark `square` four ways: discarded, sunk, via `b.Loop`, and with an empty `b.Loop` control.
2. Compare C against D. If they're equal, C measured D.
3. Batch 1000 calls into one op and divide — the only number here that means anything.
4. Repeat the discard trap on `fib(20)` to see where DCE stops.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"testing"
)

// square is small and pure, so the compiler can inline it -- and then delete it
// entirely if nobody uses the result.
func square(x int) int { return x * x }

// fib is recursive, so it cannot be inlined and the compiler will not remove it.
func fib(n int) int {
	if n < 2 {
		return n
	}
	return fib(n-1) + fib(n-2)
}

var sink int

// nsPerOp keeps the fractional part that BenchmarkResult.NsPerOp truncates to 0
// -- which is exactly the range this example lives in.
func nsPerOp(r testing.BenchmarkResult) float64 {
	return float64(r.T.Nanoseconds()) / float64(r.N)
}

func main() {
	rng := rand.New(rand.NewPCG(9, 4))
	const batch = 1000
	xs := make([]int, batch)
	for i := range xs {
		xs[i] = rng.IntN(1 << 20)
	}

	// A. The trap: the result is discarded, so the call is deleted.
	discarded := testing.Benchmark(func(b *testing.B) {
		for i := 0; i < b.N; i++ {
			square(i)
		}
	})

	// B. The "classic fix" -- accumulate into a package-level sink.
	sunk := testing.Benchmark(func(b *testing.B) {
		s := 0
		for i := 0; i < b.N; i++ {
			s += square(i)
		}
		sink = s
	})

	// C. b.Loop (Go 1.24+) keeps the call alive without a sink.
	looped := testing.Benchmark(func(b *testing.B) {
		i := 0
		for b.Loop() {
			square(i)
			i++
		}
	})

	// D. THE CONTROL. An empty b.Loop. Whatever this costs, C cannot cost less.
	control := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
		}
	})

	// E. Batch the work so one "op" is 1000 calls, then divide.
	batched := testing.Benchmark(func(b *testing.B) {
		s := 0
		for b.Loop() {
			for _, v := range xs {
				s += square(v)
			}
		}
		sink = s
	})

	// F. The same discard trap on a function the compiler cannot see through.
	fibDiscarded := testing.Benchmark(func(b *testing.B) {
		for i := 0; i < b.N; i++ {
			fib(20)
		}
	})

	fmt.Println("measuring square(x) -- one multiply:")
	fmt.Printf("  A. discarded, b.N loop      %9.4f ns/op  (%d iters)\n", nsPerOp(discarded), discarded.N)
	fmt.Printf("  B. summed into a sink       %9.4f ns/op  (%d iters)\n", nsPerOp(sunk), sunk.N)
	fmt.Printf("  C. b.Loop                   %9.4f ns/op  (%d iters)\n", nsPerOp(looped), looped.N)
	fmt.Printf("  D. b.Loop, EMPTY BODY       %9.4f ns/op  <- the control\n", nsPerOp(control))
	fmt.Printf("  E. %d per op, divided     %9.4f ns/call\n", batch, nsPerOp(batched)/batch)

	fmt.Println()
	fmt.Println("what actually happened:")
	fmt.Println("  A and B were both optimized away -- the sink did NOT save B.")
	fmt.Printf("  C (%.2f) is indistinguishable from D (%.2f): it measured the harness.\n", nsPerOp(looped), nsPerOp(control))
	fmt.Println("  only E measured square, by making one op big enough to see.")

	fmt.Println()
	fmt.Printf("but DCE is selective -- fib(20) discarded in a b.N loop: %.0f ns/op\n", nsPerOp(fibDiscarded))
	fmt.Println("recursive, non-inlinable, not provably pure -> the compiler leaves it alone.")

	fmt.Println()
	fmt.Println("rules: always run an empty control; below ~2 ns/op, batch and divide.")
}
```

**Sample output** (values vary; the A≈B≪C≈D relationship is the point):

```
measuring square(x) -- one multiply:
  A. discarded, b.N loop         0.2325 ns/op  (1000000000 iters)
  B. summed into a sink          0.2604 ns/op  (1000000000 iters)
  C. b.Loop                      1.6206 ns/op  (742929212 iters)
  D. b.Loop, EMPTY BODY          1.6042 ns/op  <- the control
  E. 1000 per op, divided        1.0504 ns/call

what actually happened:
  A and B were both optimized away -- the sink did NOT save B.
  C (1.62) is indistinguishable from D (1.60): it measured the harness.
  only E measured square, by making one op big enough to see.

but DCE is selective -- fib(20) discarded in a b.N loop: 14676 ns/op
recursive, non-inlinable, not provably pure -> the compiler leaves it alone.

rules: always run an empty control; below ~2 ns/op, batch and divide.
```

The tell in A and B is the iteration count: **1,000,000,000** is `testing`'s hard cap. A benchmark
that hits the cap and still reports a fraction of a nanosecond ran an empty loop.

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// ---- ns/op: stand-in for testing.Benchmark / b.Loop ----
// Same calibration as the kit's ns_per_op, but it also returns the iteration
// count -- in this example the count is evidence (the "tell" of a deleted body).
fn bench(mut f: impl FnMut()) -> (f64, u64) {
    let target = Duration::from_millis(200);
    let mut n: u64 = 1;
    loop {
        let t = Instant::now();
        for _ in 0..n {
            f();
        }
        let el = t.elapsed();
        if el >= target || n >= 1 << 40 {
            return (el.as_nanos() as f64 / n as f64, n);
        }
        let per = el.as_nanos().max(1) as f64 / n as f64;
        n = ((target.as_nanos() as f64 * 1.2 / per) as u64).clamp(n + 1, n * 100);
    }
}

// square is small and pure, so the compiler can inline it -- and then delete it
// entirely if nobody uses the result.
fn square(x: u64) -> u64 {
    x.wrapping_mul(x)
}

// fib is recursive, so it cannot be inlined into a loop and, unlike square,
// the compiler cannot prove it terminates.
fn fib(n: u32) -> u64 {
    if n < 2 {
        return u64::from(n);
    }
    fib(n - 1) + fib(n - 2)
}

fn main() {
    // Seeded, deterministic inputs (std has no RNG; an LCG is enough here).
    const BATCH: usize = 1000;
    let mut state = 9u64;
    let xs: Vec<u64> = (0..BATCH)
        .map(|_| {
            state = state
                .wrapping_mul(6364136223846793005)
                .wrapping_add(1442695040888963407);
            state >> 44
        })
        .collect();

    // A. The trap: the result is discarded, so the call is deleted.
    let mut i = 0u64;
    let (discarded, discarded_n) = bench(|| {
        square(i);
        i += 1;
    });

    // B. The "classic fix" -- accumulate into a sink. Rust has no package-level
    // mutable variable, so the sink is a local that black_box reads at the end.
    let mut s = 0u64;
    let mut i = 0u64;
    let (sunk, sunk_n) = bench(|| {
        s = s.wrapping_add(square(i));
        i += 1;
    });
    black_box(s);

    // C. black_box the input and the output: Rust's b.Loop. The call has to
    // happen, because the compiler must assume black_box reads and changes it.
    let mut i = 0u64;
    let (looped, looped_n) = bench(|| {
        black_box(square(black_box(i)));
        i += 1;
    });

    // D. THE CONTROL: the same black_box dance with no multiply in between.
    // Whatever this costs, C should not cost less. (An *empty* closure would be
    // useless here: the optimiser deletes it just like A, so the control has to
    // contain the black_box calls.)
    let mut i = 0u64;
    let (control, _) = bench(|| {
        black_box(black_box(i));
        i += 1;
    });

    // E. Batch the work so one "op" is 1000 calls, then divide.
    let mut s = 0u64;
    let (batched, _) = bench(|| {
        for &v in black_box(&xs) {
            s = s.wrapping_add(square(v));
        }
    });
    black_box(s);

    // F. The same discard trap on a function the compiler cannot inline into a
    // loop. F1 discards the result; F2 hides argument and result in black_box.
    let (fib_discarded, _) = bench(|| {
        fib(20);
    });
    let (fib_kept, _) = bench(|| {
        black_box(fib(black_box(20)));
    });

    println!("measuring square(x) -- one multiply:");
    println!("  A. discarded                {discarded:>9.4} ns/op  ({discarded_n} iters)");
    println!("  B. summed into a sink       {sunk:>9.4} ns/op  ({sunk_n} iters)");
    println!("  C. black_box in and out     {looped:>9.4} ns/op  ({looped_n} iters)");
    println!("  D. black_box only, NO CALL  {control:>9.4} ns/op  <- the control");
    println!(
        "  E. {BATCH} per op, divided    {:>9.4} ns/call",
        batched / BATCH as f64
    );

    println!();
    println!("what actually happened:");
    println!("  A and B finished instantly and ran the iteration cap away: the work was deleted,");
    println!("  and the sink did NOT save B.");
    println!(
        "  C ({looped:.2}) is not slower than D ({control:.2}): the multiply is invisible, so C"
    );
    println!("  measured the black_box loop, not square.");
    println!(
        "  E ({:.2} ns/call) is the only row that is not just the harness -- one op big",
        batched / BATCH as f64
    );
    println!("  enough to see. Read it as throughput (the loop may be unrolled), not latency.");

    println!();
    println!("fib(20), a recursive pure function, in the same kind of loop:");
    println!("  F1. discarded               {fib_discarded:>9.4} ns/op  <- not executed n times");
    println!("  F2. behind black_box        {fib_kept:>9.0} ns/op");

    println!();
    println!("rules: run a control that cannot be deleted; below ~1 ns/op, batch and divide;");
    println!("black_box whatever the optimiser could precompute, hoist or delete.");
}
```

**Rust sample output:**

```
measuring square(x) -- one multiply:
  A. discarded                   0.0000 ns/op  (100000000000000 iters)
  B. summed into a sink          0.0000 ns/op  (100000000000000 iters)
  C. black_box in and out        0.2649 ns/op  (914034782 iters)
  D. black_box only, NO CALL     0.3625 ns/op  <- the control
  E. 1000 per op, divided       0.1634 ns/call

what actually happened:
  A and B finished instantly and ran the iteration cap away: the work was deleted,
  and the sink did NOT save B.
  C (0.26) is not slower than D (0.36): the multiply is invisible, so C
  measured the black_box loop, not square.
  E (0.16 ns/call) is the only row that is not just the harness -- one op big
  enough to see. Read it as throughput (the loop may be unrolled), not latency.

fib(20), a recursive pure function, in the same kind of loop:
  F1. discarded                  0.0000 ns/op  <- not executed n times
  F2. behind black_box            11759 ns/op

rules: run a control that cannot be deleted; below ~1 ns/op, batch and divide;
black_box whatever the optimiser could precompute, hoist or delete.
```

> **Go vs Rust:** The lesson survives the move, with Rust-shaped details. `std::hint::black_box` plays `b.Loop`'s role, and Go's package-level `sink` becomes a local that `black_box` reads at the end — which does not save B here either (0.0000 ns/op at 10^14 iterations; a standalone copy of that loop compiles to a closed-form formula with no loop, checked with `--emit asm`). An *empty* control body would be deleted like A, so the control D has to keep the `black_box` calls, and it is only approximate: C (0.26–0.28 ns) came out *below* D (0.35–0.41 ns) in all six runs, so both are measuring the `black_box` loop (~1–1.5 cycles), not the multiply. The Go prose's claim that a recursive function is left alone does not hold for LLVM: `fib(20)` discarded measures 0.0000 ns/op, and the asm shows why — it was not deleted but hoisted out of the loop as loop-invariant, so it runs once per batch instead of n times; behind `black_box` it costs ~12 µs (Go: 14.7 µs).

**Complexity:** not about complexity at all — this is about whether your *measurement* of one is real. Revisited in depth at [42](../../42-benchmarking-profiling.md).

---

## 13. `[]T` vs `[]*T`: the pointer tax

`🔴 hard` · *Cache locality / GC scanning*

Two loops, both O(n), both summing a million points. One walks contiguous memory; the other follows a
million shuffled pointers. Same complexity, **2.7× the time** — and then a second bill arrives that
Big-O never mentions: a GC cycle with a million live pointers took **~59× longer** than with a
million pointer-free values, because the collector must trace every one of them, on every cycle.

Note the structure of the experiment: each phase builds its data **inside its own function**, so the
other structure is unreachable when the GC is timed. Measuring both while both are alive would time
the same heap twice and show no difference at all.

**Steps:**

1. Build `[]point` in one scope, benchmark the sum, time 10 GC cycles, return.
2. `runtime.GC()` to drop phase 1, then repeat with `[]*point` — shuffled, because a real structure isn't laid out in allocation order.
3. Compare both the sum time and the GC time.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"runtime"
	"testing"
	"time"
)

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

// avgGC times a full GC cycle, averaged over n runs. Whatever is reachable when
// this runs is what the collector has to walk.
func avgGC(n int) time.Duration {
	runtime.GC() // warm up: don't let the first, larger cycle skew the average
	start := time.Now()
	for i := 0; i < n; i++ {
		runtime.GC()
	}
	return time.Since(start) / time.Duration(n)
}

// Each phase builds its structure in its own scope, so that by the time the
// other phase runs, this one is unreachable and the GC numbers are comparable.

func measureValues(n int) (testing.BenchmarkResult, time.Duration) {
	values := make([]point, n)
	for i := range values {
		values[i] = point{x: i, y: i}
	}
	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			sumValues(values)
		}
	})
	gc := avgGC(10)
	runtime.KeepAlive(values)
	return res, gc
}

func measurePointers(n int) (testing.BenchmarkResult, time.Duration) {
	pointers := make([]*point, n)
	for i := range pointers {
		pointers[i] = &point{x: i, y: i}
	}
	// Shuffle: a structure built up over time is not laid out in allocation
	// order, and the CPU prefetcher can no longer guess what comes next.
	rng := rand.New(rand.NewPCG(3, 5))
	rng.Shuffle(n, func(i, j int) { pointers[i], pointers[j] = pointers[j], pointers[i] })

	res := testing.Benchmark(func(b *testing.B) {
		for b.Loop() {
			sumPointers(pointers)
		}
	})
	gc := avgGC(10)
	runtime.KeepAlive(pointers)
	return res, gc
}

func main() {
	const n = 1_000_000

	valRes, valGC := measureValues(n)
	runtime.GC() // drop phase 1 before phase 2 allocates
	ptrRes, ptrGC := measurePointers(n)

	fmt.Printf("sum of %d points\n\n", n)
	fmt.Printf("[]point   (contiguous)  %10d ns/op  %6.2f ns/element\n", valRes.NsPerOp(), float64(valRes.NsPerOp())/n)
	fmt.Printf("[]*point  (chased)      %10d ns/op  %6.2f ns/element  %.1fx slower\n",
		ptrRes.NsPerOp(), float64(ptrRes.NsPerOp())/n, float64(ptrRes.NsPerOp())/float64(valRes.NsPerOp()))

	fmt.Printf("\nGC cycle, 1M values live   %12v   (0 pointers to scan)\n", valGC.Round(time.Microsecond))
	fmt.Printf("GC cycle, 1M pointers live %12v   (%.1fx more work)\n",
		ptrGC.Round(time.Microsecond), float64(ptrGC)/float64(valGC))

	fmt.Println()
	fmt.Println("both loops are O(n). one walks memory, the other chases it.")
	fmt.Println("and every pointer is a second tax: the GC must trace it on every cycle.")
	fmt.Println("in Go, choosing []T over []*T is an algorithmic decision, not a style one.")
}
```

**Sample output** (timings vary; the ratios are stable):

```
sum of 1000000 points

[]point   (contiguous)      307238 ns/op    0.31 ns/element
[]*point  (chased)          817159 ns/op    0.82 ns/element  2.7x slower

GC cycle, 1M values live           64µs   (0 pointers to scan)
GC cycle, 1M pointers live      3.753ms   (58.8x more work)

both loops are O(n). one walks memory, the other chases it.
and every pointer is a second tax: the GC must trace it on every cycle.
in Go, choosing []T over []*T is an algorithmic decision, not a style one.
```

This is the reason Part 2 keeps saying "default to a slice", and the reason a linked list — which is
*nothing but* pointer chasing — loses to a slice far more often than the complexity table suggests.

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

// The machine is shared and timings jitter, so take the fastest of three
// calibrated runs: noise only ever adds time, never removes it.
fn best_ns_per_op(mut f: impl FnMut()) -> f64 {
    (0..3)
        .map(|_| ns_per_op(&mut f))
        .fold(f64::INFINITY, f64::min)
}

// Rng: splitmix64, standing in for math/rand (std has no RNG). Seeded.
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
    /// like rand.Shuffle
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

fn sum_boxes(ps: &[Box<Point>]) -> i64 {
    ps.iter().map(|p| p.x + p.y).sum()
}

// What one layout costs. Go's second bill is a GC cycle that re-traces every
// live pointer; Rust has no collector, so the bills it does send are
// allocator calls to build and allocator calls to free (Drop).
struct Phase {
    sum_ns: f64,
    allocs: usize,
    build: Duration,
    drop: Duration,
}

// measure builds the structure, benchmarks the sum, and times Drop. Only one
// structure is alive at a time, like Go's one-phase-per-function layout.
// Build and Drop times are the fastest of 5 repetitions.
fn measure<T>(build: impl Fn() -> T, prep: impl Fn(&mut T), sum: impl Fn(&T) -> i64) -> Phase {
    let (allocs, _, _) = allocs_of(&build);
    let mut phase = Phase {
        sum_ns: 0.0,
        allocs,
        build: Duration::MAX,
        drop: Duration::MAX,
    };
    for rep in 0..5 {
        let t = Instant::now();
        let mut data = build();
        phase.build = phase.build.min(t.elapsed());
        prep(&mut data);
        if rep == 0 {
            phase.sum_ns = best_ns_per_op(|| {
                black_box(sum(black_box(&data)));
            });
        }
        // black_box: without it the optimiser can see that `data` is built and
        // immediately dropped, and delete the whole allocate/fill/free (it did,
        // in an earlier draft: Drop "took" 0.2 microseconds).
        let data = black_box(data);
        let t = Instant::now();
        drop(data);
        phase.drop = phase.drop.min(t.elapsed());
    }
    phase
}

fn main() {
    const N: usize = 1_000_000;

    let values = measure(
        || -> Vec<Point> { (0..N as i64).map(|i| Point { x: i, y: i }).collect() },
        |_| {},
        |v| sum_values(v),
    );

    let boxes = |shuffle: bool| {
        measure(
            || -> Vec<Box<Point>> {
                (0..N as i64)
                    .map(|i| Box::new(Point { x: i, y: i }))
                    .collect()
            },
            // Shuffle: a structure built up over time is not laid out in
            // allocation order, and the CPU prefetcher can no longer guess
            // what comes next.
            |v| {
                if shuffle {
                    Rng::new(3).shuffle(v)
                }
            },
            |v| sum_boxes(v),
        )
    };
    let chased = boxes(true);
    let ordered = boxes(false);

    println!("sum of {N} points\n");
    let row = |label: &str, p: &Phase, vs_values: Option<f64>| {
        let note = vs_values.map_or(String::new(), |r| format!(" {r:.1}x slower"));
        println!(
            "{label:<30} {:>10.0} ns/op {:>6.2} ns/element{note}",
            p.sum_ns,
            p.sum_ns / N as f64
        );
    };
    row("Vec<Point>   (contiguous)", &values, None);
    row(
        "Vec<Box<Point>> (shuffled)",
        &chased,
        Some(chased.sum_ns / values.sum_ns),
    );
    row(
        "Vec<Box<Point>> (alloc order)",
        &ordered,
        Some(ordered.sum_ns / values.sum_ns),
    );

    println!("\nthe bills Rust does send (no GC cycle exists to time):");
    println!(
        "{:<30} {:>12} {:>10} {:>10} {:>10} {:>10}",
        "", "allocations", "build", "ns/elem", "Drop", "ns/elem"
    );
    for (label, p) in [
        ("Vec<Point>", &values),
        ("Vec<Box<Point>> (shuffled)", &chased),
        ("Vec<Box<Point>> (alloc order)", &ordered),
    ] {
        println!(
            "{label:<30} {:>12} {:>10.2?} {:>10.2} {:>10.2?} {:>10.2}",
            p.allocs,
            p.build,
            p.build.as_nanos() as f64 / N as f64,
            p.drop,
            p.drop.as_nanos() as f64 / N as f64
        );
    }
    println!();
    println!("Vec<Point> is one allocation and one free, however many points it holds.");
    println!("Vec<Box<Point>> is one allocation and one free *per point*.");
    println!(
        "same boxes, same count: shuffling alone costs {:.1}x on the sum and {:.1}x on Drop",
        chased.sum_ns / ordered.sum_ns,
        chased.drop.as_secs_f64() / ordered.drop.as_secs_f64()
    );
}
```

**Rust sample output:**

```
sum of 1000000 points

Vec<Point>   (contiguous)          171312 ns/op   0.17 ns/element
Vec<Box<Point>> (shuffled)         919534 ns/op   0.92 ns/element 5.4x slower
Vec<Box<Point>> (alloc order)      288205 ns/op   0.29 ns/element 1.7x slower

the bills Rust does send (no GC cycle exists to time):
                                allocations      build    ns/elem       Drop    ns/elem
Vec<Point>                                1   134.29µs       0.13   166.00ns       0.00
Vec<Box<Point>> (shuffled)          1000001    12.39ms      12.39    46.08ms      46.08
Vec<Box<Point>> (alloc order)       1000001    12.43ms      12.43    22.76ms      22.76

Vec<Point> is one allocation and one free, however many points it holds.
Vec<Box<Point>> is one allocation and one free *per point*.
same boxes, same count: shuffling alone costs 3.2x on the sum and 2.0x on Drop
```

> **Go vs Rust:** There is no GC cycle to time: no collector ever re-traces a live `Vec<Box<Point>>`, so Go's 58.8x line has no Rust counterpart. What Rust charges instead is allocator calls: 1 allocation for `Vec<Point>` vs 1,000,001 for the boxes (deterministic), roughly 12–16 ns per `Box` to build and 20–124 ns per `Box` to drop, against about a microsecond or less to free the whole `Vec<Point>`. The sum is where machine load shows: over 15 runs on a busy machine the shuffled boxes were 3.4x–14x slower than the contiguous vec (Go sample: 2.7x) and the same boxes in allocation order 0.9x–2.9x, so shuffling accounts for most of the gap (cache misses are the plausible mechanism, not measured); the shuffled row was the slowest in every run.

**Complexity:** both O(n) time, O(n) space · the difference is entirely in the constant — and in GC work the notation doesn't model at all. Full treatment at [04](../../04-go-memory-model.md).

---

## 14. The brute-force oracle

`🔴 hard` · *Differential testing*

The pattern that verifies every clever implementation in the rest of this plan. Write the obviously
correct, obviously slow version. Write the fast one. Feed both the **same small random inputs** and
assert they agree. A deliberately broken two-sum — one that lets an element pair with itself — is
caught here in **26 trials**, complete with a minimal counterexample you can paste into a unit test.

Note the design choice: the oracle compares *existence*, not indices. Two correct implementations may
legitimately return different valid pairs, so compare a property both must agree on.

**Steps:**

1. Write the O(n²) reference — it only has to be right.
2. Write the O(n) candidate, and a variant with a realistic off-by-one bug.
3. Run both against the oracle on thousands of tiny random inputs (small n and a small value range make collisions — and therefore bugs — frequent).

```go
package main

import (
	"fmt"
	"math/rand/v2"
)

// The oracle: obviously correct, obviously slow. O(n^2).
func hasPairBrute(xs []int, target int) bool {
	for i := 0; i < len(xs); i++ {
		for j := i + 1; j < len(xs); j++ {
			if xs[i]+xs[j] == target {
				return true
			}
		}
	}
	return false
}

// The candidate: fast, and correct. O(n).
func hasPairFast(xs []int, target int) bool {
	seen := make(map[int]struct{}, len(xs))
	for _, v := range xs {
		if _, ok := seen[target-v]; ok {
			return true
		}
		seen[v] = struct{}{}
	}
	return false
}

// The same idea with one bug: it records the value BEFORE looking for its
// complement, so an element can pair with itself.
func hasPairBuggy(xs []int, target int) bool {
	seen := make(map[int]struct{}, len(xs))
	for _, v := range xs {
		seen[v] = struct{}{}
		if _, ok := seen[target-v]; ok {
			return true
		}
	}
	return false
}

func check(name string, candidate func([]int, int) bool) {
	rng := rand.New(rand.NewPCG(42, 7))
	const trials = 200_000

	for t := 0; t < trials; t++ {
		xs := make([]int, rng.IntN(7))
		for i := range xs {
			xs[i] = rng.IntN(11) - 5
		}
		target := rng.IntN(11) - 5

		want := hasPairBrute(xs, target)
		got := candidate(xs, target)
		if want != got {
			fmt.Printf("%-12s FAIL after %d trials\n", name, t+1)
			fmt.Printf("             xs=%v target=%d -> oracle=%v candidate=%v\n", xs, target, want, got)
			return
		}
	}
	fmt.Printf("%-12s ok  (%d random trials agree with the oracle)\n", name, trials)
}

func main() {
	check("fast", hasPairFast)
	check("buggy", hasPairBuggy)

	fmt.Println()
	fmt.Println("small random inputs + a slow reference = the cheapest correctness proof there is")
	fmt.Println("this pattern verifies every clever implementation in the rest of the plan")
}
```

**Output** (deterministic — the PCG seed is fixed):

```
fast         ok  (200000 random trials agree with the oracle)
buggy        FAIL after 26 trials
             xs=[5 3 2] target=4 -> oracle=false candidate=true

small random inputs + a slow reference = the cheapest correctness proof there is
this pattern verifies every clever implementation in the rest of the plan
```

Read the counterexample: `[5 3 2]` with target 4 has no valid pair, but the buggy version inserts `2`,
then looks for `4-2 = 2`, finds the copy it just inserted, and reports success. **Small inputs are a
feature** — a failure on `n=3` is a failure you can debug by hand.

**Rust:**

```rust
use std::collections::HashSet;

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN. The kit's splitmix64 would give a different stream, so this example
// would find its counterexample after a different number of trials; with the
// same generator and the same seed (42, 7) it sees exactly the inputs Go sees.
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

// The oracle: obviously correct, obviously slow. O(n^2).
fn has_pair_brute(xs: &[i64], target: i64) -> bool {
    for (i, a) in xs.iter().enumerate() {
        for b in &xs[i + 1..] {
            if a + b == target {
                return true;
            }
        }
    }
    false
}

// The candidate: fast, and correct. O(n).
fn has_pair_fast(xs: &[i64], target: i64) -> bool {
    let mut seen = HashSet::with_capacity(xs.len());
    for &v in xs {
        if seen.contains(&(target - v)) {
            return true;
        }
        seen.insert(v);
    }
    false
}

// The same idea with one bug: it records the value BEFORE looking for its
// complement, so an element can pair with itself.
fn has_pair_buggy(xs: &[i64], target: i64) -> bool {
    let mut seen = HashSet::with_capacity(xs.len());
    for &v in xs {
        seen.insert(v);
        if seen.contains(&(target - v)) {
            return true;
        }
    }
    false
}

// Go prints a slice as [5 3 2]; Rust's {:?} prints [5, 3, 2]. Format by hand
// so the output reads the same in both languages.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn check(name: &str, candidate: fn(&[i64], i64) -> bool) {
    let mut rng = Pcg::new(42, 7);
    const TRIALS: u32 = 200_000;

    for t in 0..TRIALS {
        let len = rng.int_n(7) as usize;
        let xs: Vec<i64> = (0..len).map(|_| rng.int_n(11) as i64 - 5).collect();
        let target = rng.int_n(11) as i64 - 5;

        let want = has_pair_brute(&xs, target);
        let got = candidate(&xs, target);
        if want != got {
            println!("{name:<12} FAIL after {} trials", t + 1);
            println!(
                "             xs={} target={target} -> oracle={want} candidate={got}",
                go_slice(&xs)
            );
            return;
        }
    }
    println!("{name:<12} ok  ({TRIALS} random trials agree with the oracle)");
}

fn main() {
    check("fast", has_pair_fast);
    check("buggy", has_pair_buggy);

    println!();
    println!("small random inputs + a slow reference = the cheapest correctness proof there is");
    println!("this pattern verifies every clever implementation in the rest of the plan");
}
```

**Rust output:**

```
fast         ok  (200000 random trials agree with the oracle)
buggy        FAIL after 26 trials
             xs=[5 3 2] target=4 -> oracle=false candidate=true

small random inputs + a slow reference = the cheapest correctness proof there is
this pattern verifies every clever implementation in the rest of the plan
```

> **Go vs Rust:** Identical output, byte for byte (also diffed against a live run of the Go program): `Pcg` is a ~30-line port of Go's `math/rand/v2` PCG-DXSM generator and its `IntN`, so the same seed (42, 7) feeds both programs the same 26 inputs; with the kit's splitmix64 the counterexample would have appeared after a different trial count. Std has no RNG, so a hand-written generator is the Rust cost of a seeded property test, and `cargo-fuzz` / `proptest` are what you would reach for in a real project (not run here). `go_slice` prints `[5 3 2]` because Rust's `{:?}` would print `[5, 3, 2]`.

**Complexity:** oracle O(n²) · candidate O(n) · the harness is O(trials · n²), and still finishes in under a second because n ≤ 6. Full treatment, including `go test -fuzz`, at [43](../../43-testing-algorithms.md).

---

> That's the tier. Back to the [index](README.md), or on to [02 — Environment & Toolkit](../../02-environment-setup.md),
> which turns examples 7, 8, 12 and 14 into a reusable harness.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
