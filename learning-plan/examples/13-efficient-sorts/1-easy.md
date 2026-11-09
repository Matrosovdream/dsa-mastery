# Step 13 — Efficient Sorts: Divide & Conquer · 🟢 Easy

Examples **1–6**: the merge step, merge sort around it, the one character that decides stability,
the two partitions, the input that makes quicksort quadratic, and heapsort in place.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic — comparison counts and seeded inputs — and should match
yours character for character; any **Sample output:** block prints timings from one machine.

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

## 1. The merge step, and the inversions it counts

`🟢 easy` · *Merge / Θ(m + n)*

The merge is all of merge sort's real work: the recursion only decides which runs to merge, so its cost is the
cost of the merge. That cost has bounds on both sides. Two sorted runs of lengths m and n take m + n − 1
comparisons when they interleave, and only min(m, n) when one lies wholly below the other, because once a run is
used up the rest of the other is copied with no comparison at all. Time is Θ(m + n) either way. The same loop
also counts the inversions that cross the two runs, at no extra cost: each time the right head wins, it is
smaller than every element still waiting on the left, so it jumps all of them in one step.

**Steps:**

1. Write `merge(a, b)` that returns the merged slice, the comparison count and the cross-inversion count.
2. Run four fixed cases with m + n = 16 and compare the count with m + n − 1 and min(m, n).
3. Check the cross-inversion count on 1,000 seeded pairs of runs (with ties) against a brute-force double loop.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
)

// merge combines two sorted runs into one sorted slice. It is the whole of
// merge sort's work: the recursion only decides WHICH runs to merge.
//
// cmps counts comparisons: one per loop iteration, because each iteration
// compares the two heads once. When one run is used up the loop stops and the
// rest of the other run is copied without a single comparison.
//
// cross counts the inversions that cross the two runs: pairs (x in a, y in b)
// with x > y. Each time the right head wins, it is smaller than every element
// still waiting on the left -- len(a)-i of them -- so it jumps them all at once.
func merge(a, b []int) (out []int, cmps, cross int) {
	out = make([]int, 0, len(a)+len(b))
	i, j := 0, 0
	for i < len(a) && j < len(b) {
		cmps++
		if b[j] < a[i] { // right strictly smaller: take it (ties go LEFT = stable)
			out = append(out, b[j])
			cross += len(a) - i
			j++
		} else {
			out = append(out, a[i])
			i++
		}
	}
	out = append(out, a[i:]...) // the tail copy is free: no comparisons
	out = append(out, b[j:]...)
	return out, cmps, cross
}

// bruteCross counts the same pairs with a double loop, O(m*n): the oracle.
func bruteCross(a, b []int) int {
	c := 0
	for _, x := range a {
		for _, y := range b {
			if x > y {
				c++
			}
		}
	}
	return c
}

func seq(lo, step, count int) []int { // lo, lo+step, ... (count values)
	s := make([]int, count)
	for k := range s {
		s[k] = lo + k*step
	}
	return s
}

func main() {
	fmt.Println("Section 1: comparison count depends on how the runs interleave")
	cases := []struct {
		name string
		a, b []int
	}{
		{"interleaved (odds | evens)", seq(1, 2, 8), seq(2, 2, 8)},
		{"all left smaller", seq(0, 1, 8), seq(8, 1, 8)},
		{"all right smaller", seq(8, 1, 8), seq(0, 1, 8)},
		{"one element in the middle", []int{15}, seq(0, 2, 15)},
	}
	fmt.Printf("  %-27s %2s %2s %11s %7s %8s %6s\n", "case", "m", "n", "comparisons", "m+n-1", "min(m,n)", "cross")
	free := 0 // elements placed by the tail copy in the last case
	for _, c := range cases {
		out, cmps, cross := merge(c.a, c.b)
		if !slices.IsSorted(out) {
			panic("merge output not sorted")
		}
		m, n := len(c.a), len(c.b)
		fmt.Printf("  %-27s %2d %2d %11d %7d %8d %6d\n", c.name, m, n, cmps, m+n-1, min(m, n), cross)
		free = m + n - cmps // each comparison places one element; the rest are the tail
	}
	fmt.Println("  interleaving costs the maximum, one run wholly below the other the minimum;")
	fmt.Printf("  in the last case %d of the 16 elements are placed by the tail copy, with no comparison.\n", free)

	fmt.Println()
	fmt.Println("Section 2: cross-inversions from the merge vs a brute-force double loop")
	r := rand.New(rand.NewPCG(1, 2))
	run := func() []int { // a sorted run: length IntN(20), values IntN(50) so ties occur
		s := make([]int, r.IntN(20))
		for k := range s {
			s[k] = r.IntN(50)
		}
		slices.Sort(s)
		return s
	}
	const pairs = 1000
	agree, inBounds, ties := 0, 0, 0
	for p := range pairs {
		a := run()
		b := run()
		out, cmps, cross := merge(a, b)
		brute := bruteCross(a, b)
		if cross == brute && slices.IsSorted(out) {
			agree++
		}
		m, n := len(a), len(b)
		if cmps >= min(m, n) && cmps <= max(m+n-1, 0) {
			inBounds++
		}
		for _, x := range a {
			if slices.Contains(b, x) {
				ties++ // pairs with at least one value present in both runs
				break
			}
		}
		if p < 3 {
			fmt.Printf("  pair %d: m=%2d n=%2d  comparisons %2d  cross: merge %3d, brute force %3d\n",
				p+1, m, n, cmps, cross, brute)
		}
	}
	fmt.Printf("  agree: %d/%d (and every output sorted)\n", agree, pairs)
	fmt.Printf("  comparisons within [min(m,n), m+n-1]: %d/%d\n", inBounds, pairs)
	fmt.Printf("  pairs where a value appears in both runs (ties exercised): %d/%d\n", ties, pairs)
}
```

**Output:**

```
Section 1: comparison count depends on how the runs interleave
  case                         m  n comparisons   m+n-1 min(m,n)  cross
  interleaved (odds | evens)   8  8          15      15        8     28
  all left smaller             8  8           8      15        8      0
  all right smaller            8  8           8      15        8     64
  one element in the middle    1 15           9      15        1      8
  interleaving costs the maximum, one run wholly below the other the minimum;
  in the last case 7 of the 16 elements are placed by the tail copy, with no comparison.

Section 2: cross-inversions from the merge vs a brute-force double loop
  pair 1: m=15 n=14  comparisons 28  cross: merge 107, brute force 107
  pair 2: m=13 n= 8  comparisons 20  cross: merge  32, brute force  32
  pair 3: m= 2 n=16  comparisons 17  cross: merge  22, brute force  22
  agree: 1000/1000 (and every output sorted)
  comparisons within [min(m,n), m+n-1]: 1000/1000
  pairs where a value appears in both runs (ties exercised): 598/1000
```

**Rust:**

```rust
// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so that with the same seed (1, 2) this program merges exactly the runs
// the Go program merges, and every count below comes out the same.
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

    /// like rand.IntN(n): uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: u64) -> u64 {
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

/// The merge is the whole of merge sort's work: the recursion only decides
/// WHICH runs to merge.
///
/// Returns (merged, comparisons, cross-inversions). One comparison per loop
/// iteration, because each iteration compares the two heads once; when one run
/// is used up, the rest of the other is copied without a single comparison.
/// Cross-inversions are pairs (x in a, y in b) with x > y: each time the right
/// head wins it is smaller than every element still waiting on the left --
/// a.len()-i of them -- so it jumps them all at once.
fn merge(a: &[i64], b: &[i64]) -> (Vec<i64>, usize, usize) {
    let mut out = Vec::with_capacity(a.len() + b.len()); // one allocation, exact size
    let (mut i, mut j, mut cmps, mut cross) = (0, 0, 0, 0);
    while i < a.len() && j < b.len() {
        cmps += 1;
        if b[j] < a[i] {
            // right strictly smaller: take it (ties go LEFT = stable)
            out.push(b[j]);
            cross += a.len() - i;
            j += 1;
        } else {
            out.push(a[i]);
            i += 1;
        }
    }
    out.extend_from_slice(&a[i..]); // the tail copy is free: no comparisons
    out.extend_from_slice(&b[j..]);
    (out, cmps, cross)
}

/// The same pairs with a double loop, O(m*n): the oracle.
fn brute_cross(a: &[i64], b: &[i64]) -> usize {
    a.iter().map(|x| b.iter().filter(|y| x > y).count()).sum()
}

/// lo, lo+step, ... (count values)
fn seq(lo: i64, step: i64, count: i64) -> Vec<i64> {
    (0..count).map(|k| lo + k * step).collect()
}

fn main() {
    println!("Section 1: comparison count depends on how the runs interleave");
    let cases = [
        ("interleaved (odds | evens)", seq(1, 2, 8), seq(2, 2, 8)),
        ("all left smaller", seq(0, 1, 8), seq(8, 1, 8)),
        ("all right smaller", seq(8, 1, 8), seq(0, 1, 8)),
        ("one element in the middle", vec![15], seq(0, 2, 15)),
    ];
    println!(
        "  {:<27} {:>2} {:>2} {:>11} {:>7} {:>8} {:>6}",
        "case", "m", "n", "comparisons", "m+n-1", "min(m,n)", "cross"
    );
    let mut free = 0; // elements placed by the tail copy in the last case
    for (name, a, b) in &cases {
        let (out, cmps, cross) = merge(a, b);
        assert!(out.is_sorted(), "merge output not sorted");
        let (m, n) = (a.len(), b.len());
        println!(
            "  {name:<27} {m:>2} {n:>2} {cmps:>11} {:>7} {:>8} {cross:>6}",
            m + n - 1,
            m.min(n)
        );
        free = m + n - cmps; // each comparison places one element; the rest are the tail
    }
    println!("  interleaving costs the maximum, one run wholly below the other the minimum;");
    println!(
        "  in the last case {free} of the 16 elements are placed by the tail copy, with no comparison."
    );

    println!();
    println!("Section 2: cross-inversions from the merge vs a brute-force double loop");
    let mut r = Pcg::new(1, 2);
    // a sorted run: length IntN(20), values IntN(50) so ties occur.
    // A closure that mutably borrows `r`; Rust needs `mut run` because calling it mutates.
    let mut run = || {
        let len = r.int_n(20);
        let mut s: Vec<i64> = (0..len).map(|_| r.int_n(50) as i64).collect();
        s.sort_unstable(); // integers: every correct sort gives the same result
        s
    };
    const PAIRS: usize = 1000;
    let (mut agree, mut in_bounds, mut ties) = (0, 0, 0);
    for p in 0..PAIRS {
        let a = run();
        let b = run();
        let (out, cmps, cross) = merge(&a, &b);
        let brute = brute_cross(&a, &b);
        if cross == brute && out.is_sorted() {
            agree += 1;
        }
        let (m, n) = (a.len(), b.len());
        // (m + n).saturating_sub(1) is Go's max(m+n-1, 0): usize cannot go below 0
        if cmps >= m.min(n) && cmps <= (m + n).saturating_sub(1) {
            in_bounds += 1;
        }
        if a.iter().any(|x| b.contains(x)) {
            ties += 1; // pairs with at least one value present in both runs
        }
        if p < 3 {
            println!(
                "  pair {}: m={m:>2} n={n:>2}  comparisons {cmps:>2}  cross: merge {cross:>3}, brute force {brute:>3}",
                p + 1
            );
        }
    }
    println!("  agree: {agree}/{PAIRS} (and every output sorted)");
    println!("  comparisons within [min(m,n), m+n-1]: {in_bounds}/{PAIRS}");
    println!("  pairs where a value appears in both runs (ties exercised): {ties}/{PAIRS}");
}
```

**Rust output:**

```
Section 1: comparison count depends on how the runs interleave
  case                         m  n comparisons   m+n-1 min(m,n)  cross
  interleaved (odds | evens)   8  8          15      15        8     28
  all left smaller             8  8           8      15        8      0
  all right smaller            8  8           8      15        8     64
  one element in the middle    1 15           9      15        1      8
  interleaving costs the maximum, one run wholly below the other the minimum;
  in the last case 7 of the 16 elements are placed by the tail copy, with no comparison.

Section 2: cross-inversions from the merge vs a brute-force double loop
  pair 1: m=15 n=14  comparisons 28  cross: merge 107, brute force 107
  pair 2: m=13 n= 8  comparisons 20  cross: merge  32, brute force  32
  pair 3: m= 2 n=16  comparisons 17  cross: merge  22, brute force  22
  agree: 1000/1000 (and every output sorted)
  comparisons within [min(m,n), m+n-1]: 1000/1000
  pairs where a value appears in both runs (ties exercised): 598/1000
```

> **Go vs Rust:** Output identical to Go, character for character: the `Pcg` struct is a port of `math/rand/v2`'s PCG-DXSM and `IntN`, so both programs merge the same 1,000 pairs. The merge itself is the same loop; Rust's `extend_from_slice` is Go's `append(out, a[i:]...)`, and `Vec::with_capacity` is `make([]int, 0, m+n)`. The one place `usize` shows is the bound `m + n − 1` for two empty runs, written `(m + n).saturating_sub(1)` where Go writes `max(m+n-1, 0)`.

**Complexity:** time Θ(m + n) · comparisons between min(m, n) and m + n − 1 (15 interleaved, 8 when one run is wholly below the other, 9 for one element in the middle of 15) · cross-inversions counted in the same loop, matching brute force on 1000/1000 seeded pairs · space Θ(m + n) for the output.

---

## 2. Top-down merge sort, one buffer

`🟢 easy` · *Merge sort / Θ(n log n)*

Top-down merge sort splits the slice in the middle, sorts each half recursively, and merges the two halves through
a scratch buffer. The buffer is allocated **once**, by the caller, and each call works in the matching slice of
it. This sort is Θ(n log n) on every input, and its comparison count varies with the input only inside fixed
bounds. For n a power of two it is at most n lg n − n + 1. On sorted and reversed input it is exactly
(n/2) lg n, because every merge finds one half wholly below the other. On random input it stays within a few
percent of the lg(n!) floor. The program counts all of this. It also counts the cost of the common mistake, a
`make` inside every merge, and finds that the Go compiler hides part of that cost.

**Steps:**

1. Write `mergeSort(xs, buf)` with a comparison counter and a depth counter, and a second version that `make`s a
   fresh buffer in every merge.
2. Count comparisons for n = 2⁴ … 2¹⁶ on random, sorted and reversed input, next to (n/2) lg n, n lg n − n + 1
   and lg(n!).
3. Count heap allocations per sort at n = 1,024 for both versions with `testing.AllocsPerRun`.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"testing"
)

// sorter carries the counters through the recursion.
type sorter struct {
	cmps, maxDepth int
}

// mergeSort sorts xs using buf (same length) as scratch space. The buffer is
// allocated ONCE by the caller and sliced, never re-made: xs[:mid] uses
// buf[:mid], xs[mid:] uses buf[mid:].
func (s *sorter) mergeSort(xs, buf []int, depth int) {
	s.maxDepth = max(s.maxDepth, depth)
	n := len(xs)
	if n < 2 {
		return // one element is sorted: the recursion bottoms out here
	}
	mid := n / 2
	s.mergeSort(xs[:mid], buf[:mid], depth+1) // divide: the trivial split
	s.mergeSort(xs[mid:], buf[mid:], depth+1)
	s.merge(xs, mid, buf[:n]) // conquer: all the work is here
}

// merge merges the sorted halves xs[:mid] and xs[mid:] into buf, then copies
// the result back into xs. Ties take the LEFT element, so the sort is stable.
func (s *sorter) merge(xs []int, mid int, buf []int) {
	i, j, k := 0, mid, 0
	for i < mid && j < len(xs) {
		s.cmps++
		if xs[j] < xs[i] {
			buf[k] = xs[j]
			j++
		} else {
			buf[k] = xs[i]
			i++
		}
		k++
	}
	k += copy(buf[k:], xs[i:mid]) // tails: no comparisons
	copy(buf[k:], xs[j:])
	copy(xs, buf)
}

// mergeSortAlloc is the same sort with the classic mistake: a fresh scratch
// slice made inside every merge.
func mergeSortAlloc(xs []int) {
	n := len(xs)
	if n < 2 {
		return
	}
	mid := n / 2
	mergeSortAlloc(xs[:mid])
	mergeSortAlloc(xs[mid:])
	buf := make([]int, n) // one allocation per internal node of the recursion
	var s sorter
	s.merge(xs, mid, buf)
}

func permutation(r *rand.Rand, n int) []int { // Fisher-Yates over IntN
	p := make([]int, n)
	for i := range p {
		p[i] = i
	}
	for i := n - 1; i > 0; i-- {
		j := r.IntN(i + 1)
		p[i], p[j] = p[j], p[i]
	}
	return p
}

func count(xs []int) (cmps, depth int) {
	var s sorter
	s.mergeSort(xs, make([]int, len(xs)), 0)
	if !slices.IsSorted(xs) {
		panic("not sorted")
	}
	return s.cmps, s.maxDepth
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	fmt.Println("comparisons by input shape (n a power of two)")
	fmt.Printf("  %6s %8s %8s %8s %10s %10s %9s %5s\n",
		"n", "random", "sorted", "reversed", "(n/2)lg n", "n lg n-n+1", "lg(n!)", "depth")
	exactHalf, underMax, depthLg := true, true, true
	var lastRatio float64
	for lg := 4; lg <= 16; lg += 2 {
		n := 1 << lg
		rnd, depth := count(permutation(r, n))
		sorted := make([]int, n)
		reversed := make([]int, n)
		for i := range n {
			sorted[i], reversed[i] = i, n-1-i
		}
		srt, _ := count(sorted)
		rev, _ := count(reversed)
		lgFact := 0.0 // lg(n!) = sum of log2(i), the comparison-sort lower bound
		for i := 2; i <= n; i++ {
			lgFact += math.Log2(float64(i))
		}
		half, upper := n/2*lg, n*lg-n+1
		fmt.Printf("  %6d %8d %8d %8d %10d %10d %9.0f %5d\n", n, rnd, srt, rev, half, upper, lgFact, depth)
		exactHalf = exactHalf && srt == half && rev == half
		underMax = underMax && rnd <= upper
		depthLg = depthLg && depth == lg
		lastRatio = float64(rnd) / lgFact
	}
	fmt.Println("  sorted and reversed cost exactly (n/2)lg n on every row:", exactHalf)
	fmt.Println("  random stays at or under n lg n - n + 1 on every row:", underMax)
	fmt.Printf("  random at n = 65536 is %.3f x the lg(n!) floor\n", lastRatio)
	fmt.Println("  max recursion depth = lg n on every row (root 0, single elements at the bottom):", depthLg)

	const n = 1024
	pristine := permutation(r, n)
	work := make([]int, n)
	once := testing.AllocsPerRun(100, func() {
		copy(work, pristine) // copying into an existing slice allocates nothing
		var s sorter
		s.mergeSort(work, make([]int, len(work)), 0)
	})
	each := testing.AllocsPerRun(100, func() {
		copy(work, pristine)
		mergeSortAlloc(work)
	})
	fmt.Println()
	fmt.Printf("heap allocations per sort, n = %d (testing.AllocsPerRun)\n", n)
	fmt.Printf("  buffer allocated once:  %4.0f\n", once)
	fmt.Printf("  buffer made per merge:  %4.0f   (internal nodes of the recursion: n-1 = %d)\n", each, n-1)
	fmt.Printf("  merges whose buffer did NOT reach the heap: %.0f\n", float64(n-1)-each)
	fmt.Printf("  merges of at most 4 ints (32 bytes): %d -- since Go 1.25 a non-escaping make\n", smallMerges(n, 4))
	fmt.Println("  of up to 32 bytes gets a stack buffer, so only the larger merges pay a heap allocation")
}

// smallMerges counts the merges of at most limit elements in a sort of n.
func smallMerges(n, limit int) int {
	if n < 2 {
		return 0
	}
	c := smallMerges(n/2, limit) + smallMerges(n-n/2, limit)
	if n <= limit {
		c++
	}
	return c
}
```

**Output:**

```
comparisons by input shape (n a power of two)
       n   random   sorted reversed  (n/2)lg n n lg n-n+1    lg(n!) depth
      16       45       32       32         32         49        44     4
      64      304      192      192        192        321       296     6
     256     1724     1024     1024       1024       1793      1684     8
    1024     8971     5120     5120       5120       9217      8769    10
    4096    44015    24576    24576      24576      45057     43250    12
   16384   208525   114688   114688     114688     212993    205747    14
   65536   965519   524288   524288     524288     983041    954037    16
  sorted and reversed cost exactly (n/2)lg n on every row: true
  random stays at or under n lg n - n + 1 on every row: true
  random at n = 65536 is 1.012 x the lg(n!) floor
  max recursion depth = lg n on every row (root 0, single elements at the bottom): true

heap allocations per sort, n = 1024 (testing.AllocsPerRun)
  buffer allocated once:     1
  buffer made per merge:   255   (internal nodes of the recursion: n-1 = 1023)
  merges whose buffer did NOT reach the heap: 768
  merges of at most 4 ints (32 bytes): 768 -- since Go 1.25 a non-escaping make
  of up to 32 bytes gets a stack buffer, so only the larger merges pay a heap allocation
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// ---- allocation counter: Rust's stand-in for testing.AllocsPerRun ----
// Every heap allocation in the process goes through this allocator, so the
// count is exact and deterministic (Go's AllocsPerRun is an average).
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
}
#[global_allocator]
static GLOBAL: Counting = Counting;

fn allocs_of(f: impl FnOnce()) -> usize {
    let a0 = ALLOCS.load(Relaxed);
    f();
    ALLOCS.load(Relaxed) - a0
}

// Pcg: port of Go's math/rand/v2 PCG-DXSM and IntN -- same seed, same stream.
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
    fn int_n(&mut self, n: u64) -> u64 {
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

/// Carries the counters through the recursion.
#[derive(Default)]
struct Sorter {
    cmps: usize,
    max_depth: usize,
}

impl Sorter {
    /// Sorts xs using buf (same length) as scratch. The buffer is allocated ONCE
    /// by the caller; split_at_mut hands each half its own disjoint &mut piece
    /// of both slices, which is how the borrow checker accepts the recursion.
    fn merge_sort(&mut self, xs: &mut [i64], buf: &mut [i64], depth: usize) {
        self.max_depth = self.max_depth.max(depth);
        let n = xs.len();
        if n < 2 {
            return; // one element is sorted: the recursion bottoms out here
        }
        let mid = n / 2;
        {
            let (xl, xr) = xs.split_at_mut(mid); // divide: the trivial split
            let (bl, br) = buf.split_at_mut(mid);
            self.merge_sort(xl, bl, depth + 1);
            self.merge_sort(xr, br, depth + 1);
        }
        self.merge(xs, mid, buf); // conquer: all the work is here
    }

    /// Merges xs[..mid] and xs[mid..] into buf, then copies back into xs.
    /// Ties take the LEFT element, so the sort is stable.
    fn merge(&mut self, xs: &mut [i64], mid: usize, buf: &mut [i64]) {
        let n = xs.len();
        let (mut i, mut j, mut k) = (0, mid, 0);
        while i < mid && j < n {
            self.cmps += 1;
            if xs[j] < xs[i] {
                buf[k] = xs[j];
                j += 1;
            } else {
                buf[k] = xs[i];
                i += 1;
            }
            k += 1;
        }
        // tails: no comparisons
        buf[k..k + mid - i].copy_from_slice(&xs[i..mid]);
        k += mid - i;
        buf[k..n].copy_from_slice(&xs[j..]);
        xs.copy_from_slice(&buf[..n]);
    }
}

/// The classic mistake: a fresh scratch Vec made inside every merge.
fn merge_sort_alloc(xs: &mut [i64]) {
    let n = xs.len();
    if n < 2 {
        return;
    }
    let mid = n / 2;
    merge_sort_alloc(&mut xs[..mid]);
    merge_sort_alloc(&mut xs[mid..]);
    let mut buf = vec![0; n]; // one heap allocation per internal node of the recursion
    Sorter::default().merge(xs, mid, &mut buf);
}

fn permutation(r: &mut Pcg, n: usize) -> Vec<i64> {
    // Fisher-Yates over IntN, in the same order as the Go program
    let mut p: Vec<i64> = (0..n as i64).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i as u64 + 1) as usize;
        p.swap(i, j);
    }
    p
}

fn count(mut xs: Vec<i64>) -> (usize, usize) {
    let mut s = Sorter::default();
    let mut buf = vec![0; xs.len()];
    s.merge_sort(&mut xs, &mut buf, 0);
    assert!(xs.is_sorted(), "not sorted");
    (s.cmps, s.max_depth)
}

/// Counts the merges of at most `limit` elements in a sort of n.
fn small_merges(n: usize, limit: usize) -> usize {
    if n < 2 {
        return 0;
    }
    small_merges(n / 2, limit) + small_merges(n - n / 2, limit) + usize::from(n <= limit)
}

fn main() {
    let mut r = Pcg::new(1, 2);
    println!("comparisons by input shape (n a power of two)");
    println!(
        "  {:>6} {:>8} {:>8} {:>8} {:>10} {:>10} {:>9} {:>5}",
        "n", "random", "sorted", "reversed", "(n/2)lg n", "n lg n-n+1", "lg(n!)", "depth"
    );
    let (mut exact_half, mut under_max, mut depth_lg) = (true, true, true);
    let mut last_ratio = 0.0;
    for lg in (4..=16).step_by(2) {
        let n = 1usize << lg;
        let (rnd, depth) = count(permutation(&mut r, n));
        let (srt, _) = count((0..n as i64).collect());
        let (rev, _) = count((0..n as i64).rev().collect());
        // lg(n!) = sum of log2(i), summed in the same order as Go
        let lg_fact: f64 = (2..=n).map(|i| (i as f64).log2()).sum();
        let (half, upper) = (n / 2 * lg, n * lg - n + 1);
        println!(
            "  {n:>6} {rnd:>8} {srt:>8} {rev:>8} {half:>10} {upper:>10} {lg_fact:>9.0} {depth:>5}"
        );
        exact_half &= srt == half && rev == half;
        under_max &= rnd <= upper;
        depth_lg &= depth == lg;
        last_ratio = rnd as f64 / lg_fact;
    }
    println!("  sorted and reversed cost exactly (n/2)lg n on every row: {exact_half}");
    println!("  random stays at or under n lg n - n + 1 on every row: {under_max}");
    println!("  random at n = 65536 is {last_ratio:.3} x the lg(n!) floor");
    println!(
        "  max recursion depth = lg n on every row (root 0, single elements at the bottom): {depth_lg}"
    );

    const N: usize = 1024;
    let pristine = permutation(&mut r, N);
    let mut work = vec![0; N];
    let once = allocs_of(|| {
        work.copy_from_slice(&pristine); // copying into an existing Vec allocates nothing
        let mut buf = vec![0; work.len()];
        Sorter::default().merge_sort(&mut work, &mut buf, 0);
    });
    let each = allocs_of(|| {
        work.copy_from_slice(&pristine);
        merge_sort_alloc(&mut work);
    });
    println!();
    println!("heap allocations per sort, n = {N} (counting global allocator)");
    println!("  buffer allocated once:  {once:>4}");
    println!(
        "  buffer made per merge:  {each:>4}   (internal nodes of the recursion: n-1 = {})",
        N - 1
    );
    println!(
        "  merges whose buffer did NOT reach the heap: {}",
        N - 1 - each
    );
    println!(
        "  merges of at most 4 ints (32 bytes): {} -- Rust's vec![0; n] always allocates on",
        small_merges(N, 4)
    );
    println!(
        "  the heap (n > 0), however small n is; a stack buffer has to be an array of fixed size"
    );
}
```

**Rust output:**

```
comparisons by input shape (n a power of two)
       n   random   sorted reversed  (n/2)lg n n lg n-n+1    lg(n!) depth
      16       45       32       32         32         49        44     4
      64      304      192      192        192        321       296     6
     256     1724     1024     1024       1024       1793      1684     8
    1024     8971     5120     5120       5120       9217      8769    10
    4096    44015    24576    24576      24576      45057     43250    12
   16384   208525   114688   114688     114688     212993    205747    14
   65536   965519   524288   524288     524288     983041    954037    16
  sorted and reversed cost exactly (n/2)lg n on every row: true
  random stays at or under n lg n - n + 1 on every row: true
  random at n = 65536 is 1.012 x the lg(n!) floor
  max recursion depth = lg n on every row (root 0, single elements at the bottom): true

heap allocations per sort, n = 1024 (counting global allocator)
  buffer allocated once:     1
  buffer made per merge:  1023   (internal nodes of the recursion: n-1 = 1023)
  merges whose buffer did NOT reach the heap: 0
  merges of at most 4 ints (32 bytes): 768 -- Rust's vec![0; n] always allocates on
  the heap (n > 0), however small n is; a stack buffer has to be an array of fixed size
```

> **Go vs Rust:** The comparison table is identical to Go's, row by row: same PCG permutations, same merge. The allocation section differs, and that is the finding. Rust makes **1,023** heap allocations with a buffer per merge, one per internal node. Go made **255**, because its compiler (1.25+) gives a non-escaping variable-size `make` of ≤ 32 bytes a stack buffer. Rust has no such rule: a `Vec` with capacity lives on the heap, and only a fixed-size array lives on the stack. Its counting `#[global_allocator]` gives an exact count per call, where `testing.AllocsPerRun` averages over runs. `split_at_mut` is how the recursion hands each half a disjoint `&mut` piece of both `xs` and `buf`.

**Complexity:** Θ(n log n) on every input · comparisons ≤ n lg n − n + 1, exactly (n/2) lg n on sorted and reversed input, 1.012 × lg(n!) on random input at n = 65,536 · recursion depth lg n · Θ(n) extra space · heap allocations: 1 with the buffer passed down, against 255 with a `make` per merge. It would be n − 1 = 1,023, but since Go 1.25 the compiler puts a non-escaping variable-size `make` of ≤ 32 bytes on the stack (`VariableMakeThreshold = 32` in `cmd/compile`, read in the source), so the 768 merges of 2 or 4 ints never reach the heap.

---

## 3. Stability is one character

`🟢 easy` · *Stability / the merge tie rule*

In merge sort, stability comes down to one test in the merge. If the left head is taken when the two heads are
equal (`left <= right`), equal keys keep their input order and the sort is stable. If the test is written `<` on
the wrong side, ties go to the right. The result is still sorted, and every test on distinct keys still passes,
but it is no longer stable. Lesson 12 showed why stability matters. This example shows where it lives in a fast
sort, and which standard-library calls promise it. Stability costs a merge sort nothing: both versions are the
same Θ(n log n) loop.

**Steps:**

1. Make 16 records `key:tag` with keys `IntN(4)` and tags `a`…`p` in input order, and sort them with both versions
   of the merge.
2. Count the equal-key pairs whose tags come out of order. 0 means stable.
3. Do the same for `slices.SortFunc` and `slices.SortStableFunc` at n = 200, then over 1,000 inputs of n = 12.

```go
package main

import (
	"cmp"
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
)

// rec is a record sorted by key; tag records the input order.
type rec struct{ key, tag int }

// mergeSort sorts by key. The ONLY difference between the two versions is the
// test that decides which head to take:
//
//	stable:   take the right head only if right < left  -> ties go LEFT
//	unstable: take the left head only if left < right   -> ties go RIGHT
//
// Both are correct sorts. Only the first keeps equal keys in input order.
func mergeSort(xs, buf []rec, stable bool, cmps *int) {
	n := len(xs)
	if n < 2 {
		return
	}
	mid := n / 2
	mergeSort(xs[:mid], buf[:mid], stable, cmps)
	mergeSort(xs[mid:], buf[mid:], stable, cmps)
	i, j, k := 0, mid, 0
	for i < mid && j < n {
		*cmps++
		var takeRight bool
		if stable {
			takeRight = xs[j].key < xs[i].key // left <= right keeps the left one
		} else {
			takeRight = !(xs[i].key < xs[j].key) // "<": on a tie the right one wins
		}
		if takeRight {
			buf[k] = xs[j]
			j++
		} else {
			buf[k] = xs[i]
			i++
		}
		k++
	}
	k += copy(buf[k:], xs[i:mid])
	copy(buf[k:], xs[j:n])
	copy(xs, buf[:n])
}

// reordered counts pairs of equal keys whose tags came out in the wrong order.
// 0 means the sort was stable on this input.
func reordered(xs []rec) int {
	c := 0
	for i := range xs {
		for j := i + 1; j < len(xs); j++ {
			if xs[i].key == xs[j].key && xs[i].tag > xs[j].tag {
				c++
			}
		}
	}
	return c
}

func show(xs []rec) string {
	parts := make([]string, len(xs))
	for i, x := range xs {
		parts[i] = fmt.Sprintf("%d:%c", x.key, 'a'+x.tag)
	}
	return strings.Join(parts, " ")
}

func records(r *rand.Rand, n, keys int) []rec {
	xs := make([]rec, n)
	for i := range xs {
		xs[i] = rec{r.IntN(keys), i}
	}
	return xs
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	in := records(r, 16, 4)
	fmt.Println("16 records key:tag, keys IntN(4), tags in input order")
	fmt.Println("  input    ", show(in))
	pairs := 0 // pairs of records with equal keys: the most a sort can reorder
	for i := range in {
		for j := i + 1; j < len(in); j++ {
			if in[i].key == in[j].key {
				pairs++
			}
		}
	}
	for _, v := range []struct {
		name   string
		stable bool
	}{{"\"<=\" merge", true}, {"\"<\"  merge", false}} {
		xs := slices.Clone(in)
		cmps := 0
		mergeSort(xs, make([]rec, len(xs)), v.stable, &cmps)
		sorted := slices.IsSortedFunc(xs, func(a, b rec) int { return cmp.Compare(a.key, b.key) })
		fmt.Printf("  %s %s\n", v.name, show(xs))
		fmt.Printf("             sorted: %v  stable: %v  reordered pairs: %d  comparisons: %d\n",
			sorted, reordered(xs) == 0, reordered(xs), cmps)
	}
	fmt.Printf("  equal-key pairs in the input: %d -- with ties always going right, every merge\n", pairs)
	fmt.Println("  puts the later of two equal records first, so equal keys come out in reverse order")

	byKey := func(a, b rec) int { return cmp.Compare(a.key, b.key) }
	fmt.Println()
	fmt.Println("standard library, equal-key pairs reordered (0 = stable on this input)")
	big := records(r, 200, 5)
	u, s := slices.Clone(big), slices.Clone(big)
	slices.SortFunc(u, byKey)
	slices.SortStableFunc(s, byKey)
	fmt.Printf("  n = 200, keys IntN(5):  SortFunc %4d   SortStableFunc %d\n", reordered(u), reordered(s))

	const trials = 1000
	totalU, totalS := 0, 0
	for range trials {
		small := records(r, 12, 5)
		u, s := slices.Clone(small), slices.Clone(small)
		slices.SortFunc(u, byKey)
		slices.SortStableFunc(s, byKey)
		totalU += reordered(u)
		totalS += reordered(s)
	}
	fmt.Printf("  n = 12, %d inputs:    SortFunc %4d   SortStableFunc %d\n", trials, totalU, totalS)
	fmt.Println("  at n <= 12 SortFunc runs insertion sort, which never moves an element past an equal one:")
	fmt.Println("  small inputs are stable by accident; do not rely on it")
}
```

**Output:**

```
16 records key:tag, keys IntN(4), tags in input order
  input     0:a 0:b 0:c 2:d 0:e 0:f 1:g 2:h 1:i 0:j 3:k 2:l 3:m 1:n 2:o 3:p
  "<=" merge 0:a 0:b 0:c 0:e 0:f 0:j 1:g 1:i 1:n 2:d 2:h 2:l 2:o 3:k 3:m 3:p
             sorted: true  stable: true  reordered pairs: 0  comparisons: 41
  "<"  merge 0:j 0:f 0:e 0:c 0:b 0:a 1:n 1:i 1:g 2:o 2:l 2:h 2:d 3:p 3:m 3:k
             sorted: true  stable: false  reordered pairs: 27  comparisons: 45
  equal-key pairs in the input: 27 -- with ties always going right, every merge
  puts the later of two equal records first, so equal keys come out in reverse order

standard library, equal-key pairs reordered (0 = stable on this input)
  n = 200, keys IntN(5):  SortFunc 1961   SortStableFunc 0
  n = 12, 1000 inputs:    SortFunc    0   SortStableFunc 0
  at n <= 12 SortFunc runs insertion sort, which never moves an element past an equal one:
  small inputs are stable by accident; do not rely on it
```

**Rust:**

```rust
// Pcg: port of Go's math/rand/v2 PCG-DXSM and IntN -- same seed, same records.
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
    fn int_n(&mut self, n: u64) -> u64 {
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

/// A record sorted by key; tag records the input order. Copy, so the merge can
/// move records between slices by value, as Go does.
#[derive(Clone, Copy, Default)]
struct Rec {
    key: u64,
    tag: usize,
}

/// Sorts by key. The ONLY difference between the two versions is the test
/// that decides which head to take:
///   stable:   take the right head only if right < left  -> ties go LEFT
///   unstable: take the left head only if left < right   -> ties go RIGHT
/// Both are correct sorts. Only the first keeps equal keys in input order.
fn merge_sort(xs: &mut [Rec], buf: &mut [Rec], stable: bool, cmps: &mut usize) {
    let n = xs.len();
    if n < 2 {
        return;
    }
    let mid = n / 2;
    {
        let (xl, xr) = xs.split_at_mut(mid);
        let (bl, br) = buf.split_at_mut(mid);
        merge_sort(xl, bl, stable, cmps);
        merge_sort(xr, br, stable, cmps);
    }
    let (mut i, mut j, mut k) = (0, mid, 0);
    while i < mid && j < n {
        *cmps += 1;
        let take_right = if stable {
            xs[j].key < xs[i].key // left <= right keeps the left one
        } else {
            !(xs[i].key < xs[j].key) // "<": on a tie the right one wins
        };
        if take_right {
            buf[k] = xs[j];
            j += 1;
        } else {
            buf[k] = xs[i];
            i += 1;
        }
        k += 1;
    }
    buf[k..k + mid - i].copy_from_slice(&xs[i..mid]);
    k += mid - i;
    buf[k..n].copy_from_slice(&xs[j..n]);
    xs.copy_from_slice(&buf[..n]);
}

/// Pairs of equal keys whose tags came out in the wrong order (0 = stable here).
fn reordered(xs: &[Rec]) -> usize {
    xs.iter()
        .enumerate()
        .map(|(i, a)| {
            xs[i + 1..]
                .iter()
                .filter(|b| a.key == b.key && a.tag > b.tag)
                .count()
        })
        .sum()
}

fn show(xs: &[Rec]) -> String {
    let parts: Vec<String> = xs
        .iter()
        .map(|x| format!("{}:{}", x.key, char::from(b'a' + x.tag as u8)))
        .collect();
    parts.join(" ")
}

fn records(r: &mut Pcg, n: usize, keys: u64) -> Vec<Rec> {
    (0..n)
        .map(|tag| Rec {
            key: r.int_n(keys),
            tag,
        })
        .collect()
}

fn main() {
    let mut r = Pcg::new(1, 2);
    let input = records(&mut r, 16, 4);
    println!("16 records key:tag, keys IntN(4), tags in input order");
    println!("  input     {}", show(&input));
    // pairs of records with equal keys: the most a sort can reorder
    let pairs: usize = (0..input.len())
        .map(|i| {
            input[i + 1..]
                .iter()
                .filter(|b| b.key == input[i].key)
                .count()
        })
        .sum();
    for (name, stable) in [("\"<=\" merge", true), ("\"<\"  merge", false)] {
        let mut xs = input.clone();
        let mut cmps = 0;
        let mut buf = vec![Rec::default(); xs.len()];
        merge_sort(&mut xs, &mut buf, stable, &mut cmps);
        let sorted = xs.is_sorted_by_key(|x| x.key);
        println!("  {name} {}", show(&xs));
        println!(
            "             sorted: {sorted}  stable: {}  reordered pairs: {}  comparisons: {cmps}",
            reordered(&xs) == 0,
            reordered(&xs)
        );
    }
    println!(
        "  equal-key pairs in the input: {pairs} -- with ties always going right, every merge"
    );
    println!(
        "  puts the later of two equal records first, so equal keys come out in reverse order"
    );

    // In Rust the STABLE sort has the short name: sort / sort_by_key.
    // The unstable one must be asked for: sort_unstable / sort_unstable_by_key.
    println!();
    println!("standard library, equal-key pairs reordered (0 = stable on this input)");
    let big = records(&mut r, 200, 5);
    let (mut u, mut s) = (big.clone(), big.clone());
    u.sort_unstable_by_key(|x| x.key);
    s.sort_by_key(|x| x.key);
    println!(
        "  n = 200, keys IntN(5):  sort_unstable_by_key {:>4}   sort_by_key {}",
        reordered(&u),
        reordered(&s)
    );

    const TRIALS: usize = 1000;
    let (mut total_u, mut total_s) = (0, 0);
    for _ in 0..TRIALS {
        let small = records(&mut r, 12, 5);
        let (mut u, mut s) = (small.clone(), small.clone());
        u.sort_unstable_by_key(|x| x.key);
        s.sort_by_key(|x| x.key);
        total_u += reordered(&u);
        total_s += reordered(&s);
    }
    println!(
        "  n = 12, {TRIALS} inputs:    sort_unstable_by_key {total_u:>4}   sort_by_key {total_s}"
    );
    println!(
        "  at n <= 20 sort_unstable_by_key runs insertion sort, which never moves an element past an equal one:"
    );
    println!("  small inputs are stable by accident; do not rely on it");
}
```

**Rust output:**

```
16 records key:tag, keys IntN(4), tags in input order
  input     0:a 0:b 0:c 2:d 0:e 0:f 1:g 2:h 1:i 0:j 3:k 2:l 3:m 1:n 2:o 3:p
  "<=" merge 0:a 0:b 0:c 0:e 0:f 0:j 1:g 1:i 1:n 2:d 2:h 2:l 2:o 3:k 3:m 3:p
             sorted: true  stable: true  reordered pairs: 0  comparisons: 41
  "<"  merge 0:j 0:f 0:e 0:c 0:b 0:a 1:n 1:i 1:g 2:o 2:l 2:h 2:d 3:p 3:m 3:k
             sorted: true  stable: false  reordered pairs: 27  comparisons: 45
  equal-key pairs in the input: 27 -- with ties always going right, every merge
  puts the later of two equal records first, so equal keys come out in reverse order

standard library, equal-key pairs reordered (0 = stable on this input)
  n = 200, keys IntN(5):  sort_unstable_by_key 1465   sort_by_key 0
  n = 12, 1000 inputs:    sort_unstable_by_key    0   sort_by_key 0
  at n <= 20 sort_unstable_by_key runs insertion sort, which never moves an element past an equal one:
  small inputs are stable by accident; do not rely on it
```

> **Go vs Rust:** The hand-written merge sorts give output identical to Go's: the same records and the same 41 and 45 comparisons. The standard-library lines differ by design. At n = 200, Rust's `sort_unstable_by_key` (ipnsort) reordered 1,465 pairs and Go's `SortFunc` (pdqsort) 1,961: different algorithms, both unstable. Both are 0 at n = 12. Rust's cutoff is `MAX_LEN_ALWAYS_INSERTION_SORT = 20` in `core/src/slice/sort/unstable/mod.rs` (read in the source), against Go's `maxInsertion = 12`. The names are reversed between the languages: in Rust the stable sort is the short name (`sort_by_key`), and the unstable one says so (`sort_unstable_by_key`).

**Complexity:** both merges are Θ(n log n), so stability costs nothing: 41 comparisons for `<=` and 45 for `<` here. The difference comes only from which run empties first. The `<` version reorders all 27 equal-key pairs, because it reverses every group of equal keys. `slices.SortFunc` reordered 1,961 pairs at n = 200 and none at n ≤ 12, where pdqsort hands the whole range to a stable insertion sort (`maxInsertion = 12`).

---

## 4. Lomuto and Hoare partitions

`🟢 easy` · *Partition / Θ(n)*

A partition moves the elements smaller than the pivot to the left and the larger ones to the right in one Θ(n)
pass. It is quicksort's real work, as the merge is merge sort's. The two classic schemes differ in what they do
with each element. **Lomuto** scans with one index and takes the last element as pivot. It makes exactly n − 1
comparisons, swaps every element ≤ the pivot into place (about n/2 swaps), and sends every key equal to the
pivot to the same side. **Hoare** (CLRS `HOARE-PARTITION`, first element as pivot) moves two indices toward each
other and swaps only pairs that are both on the wrong side, about n/6 swaps. Because both indices stop on keys
equal to the pivot, Hoare splits a run of duplicates down the middle.

**Steps:**

1. Write both partitions with comparison and swap counters and a trace switch. In Hoare, `i` and `j` start one
   step outside the range.
2. Trace both on `[5 3 8 1 9 2 7 4 6]`, printing the array after every swap.
3. Average comparisons and swaps over 1,000 random permutations of n = 1,000, then partition 1,000 equal keys.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
)

type counts struct{ cmps, swaps int }

// lomuto partitions a[lo..hi] around the LAST element. One index j scans left
// to right; s is the store index: a[lo:s] holds the elements <= pivot found so
// far. Every such element is swapped into place -- even when s == j, which
// swaps an element with itself (CLRS does too, and we count it).
// Returns the pivot's final index p: a[lo:p] <= a[p] < a[p+1:hi+1].
func lomuto(a []int, lo, hi int, c *counts, trace bool) int {
	pivot := a[hi]
	s := lo
	for j := lo; j < hi; j++ {
		c.cmps++
		if a[j] <= pivot {
			a[s], a[j] = a[j], a[s]
			c.swaps++
			if trace {
				fmt.Printf("    swap a[%d] a[%d] -> %v\n", s, j, a)
			}
			s++
		}
	}
	a[s], a[hi] = a[hi], a[s] // pivot to its final place
	c.swaps++
	if trace {
		fmt.Printf("    swap a[%d] a[%d] -> %v  (pivot placed)\n", s, hi, a)
	}
	return s
}

// hoare is CLRS HOARE-PARTITION: pivot = FIRST element, i and j start one
// step OUTSIDE the range and move toward each other; each stops on an element
// that belongs to the other side (>= pivot for i, <= pivot for j) -- so both
// stop on keys EQUAL to the pivot. Returns j: a[lo..j] <= a[j+1..hi].
// The pivot is NOT necessarily at index j.
func hoare(a []int, lo, hi int, c *counts, trace bool) int {
	pivot := a[lo]
	i, j := lo-1, hi+1 // fine with Go's signed int; see the Rust twin for usize
	for {
		for { // repeat j-- until a[j] <= pivot
			j--
			c.cmps++
			if a[j] <= pivot {
				break
			}
		}
		for { // repeat i++ until a[i] >= pivot
			i++
			c.cmps++
			if a[i] >= pivot {
				break
			}
		}
		if i >= j {
			return j
		}
		a[i], a[j] = a[j], a[i]
		c.swaps++
		if trace {
			fmt.Printf("    swap a[%d] a[%d] -> %v\n", i, j, a)
		}
	}
}

func permutation(r *rand.Rand, n int) []int { // Fisher-Yates over IntN
	p := make([]int, n)
	for i := range p {
		p[i] = i
	}
	for i := n - 1; i > 0; i-- {
		j := r.IntN(i + 1)
		p[i], p[j] = p[j], p[i]
	}
	return p
}

func main() {
	in := []int{5, 3, 8, 1, 9, 2, 7, 4, 6}
	fmt.Println("Section 1: trace on", in)
	a := slices.Clone(in)
	var lc counts
	fmt.Println("  Lomuto, pivot = last =", a[len(a)-1])
	p := lomuto(a, 0, len(a)-1, &lc, true)
	fmt.Printf("  pivot at index %d: left %v | %d | right %v  comparisons %d, swaps %d\n",
		p, a[:p], a[p], a[p+1:], lc.cmps, lc.swaps)
	a = slices.Clone(in)
	var hc counts
	fmt.Println("  Hoare, pivot = first =", a[0])
	j := hoare(a, 0, len(a)-1, &hc, true)
	fmt.Printf("  split after index %d: left %v | right %v  comparisons %d, swaps %d\n",
		j, a[:j+1], a[j+1:], hc.cmps, hc.swaps)

	const n, trials = 1000, 1000
	r := rand.New(rand.NewPCG(1, 2))
	var lt, ht counts
	okL, okH := 0, 0
	for range trials {
		perm := permutation(r, n)
		a := slices.Clone(perm)
		p := lomuto(a, 0, n-1, &lt, false)
		if slices.Max(a[:p+1]) == a[p] && (p == n-1 || slices.Min(a[p:]) == a[p]) {
			okL++
		}
		a = slices.Clone(perm)
		j := hoare(a, 0, n-1, &ht, false)
		if slices.Max(a[:j+1]) <= slices.Min(a[j+1:]) {
			okH++
		}
	}
	avg := func(x int) float64 { return float64(x) / trials }
	fmt.Println()
	fmt.Printf("Section 2: average per partition, %d random permutations of n = %d\n", trials, n)
	fmt.Printf("  %-7s %11s %8s   %s\n", "scheme", "comparisons", "swaps", "theory")
	fmt.Printf("  %-7s %11.1f %8.1f   comparisons n-1 = %d, swaps (n-1)/2+1 = %.1f\n",
		"Lomuto", avg(lt.cmps), avg(lt.swaps), n-1, float64(n-1)/2+1)
	fmt.Printf("  %-7s %11.1f %8.1f   comparisons n+O(1), swaps (n-2)/6 = %.1f\n",
		"Hoare", avg(ht.cmps), avg(ht.swaps), float64(n-2)/6)
	fmt.Printf("  Hoare swaps / Lomuto swaps = %.3f; partition property held: Lomuto %d/%d, Hoare %d/%d\n",
		float64(ht.swaps)/float64(lt.swaps), okL, trials, okH, trials)
	// Lomuto's swap count is the pivot's rank, uniform on 1..n: variance (n*n-1)/12.
	fmt.Printf("  (Lomuto swaps = the pivot's rank, uniform on 1..n; a %d-trial mean has standard error %.1f)\n",
		trials, math.Sqrt(float64(n*n-1)/12/trials))

	fmt.Println()
	fmt.Printf("Section 3: %d equal keys\n", n)
	eq := make([]int, n)
	var le, he counts
	p = lomuto(slices.Clone(eq), 0, n-1, &le, false)
	j = hoare(slices.Clone(eq), 0, n-1, &he, false)
	fmt.Printf("  Lomuto: split %d | pivot | %d   comparisons %d, swaps %d\n", p, n-1-p, le.cmps, le.swaps)
	fmt.Printf("  Hoare:  split %d | %d          comparisons %d, swaps %d\n", j+1, n-1-j, he.cmps, he.swaps)
	fmt.Println("  Lomuto sends every equal key left (the next range is n-1 long: quadratic overall);")
	fmt.Println("  Hoare stops on equal keys from both ends and swaps its way to the middle.")
}
```

**Output:**

```
Section 1: trace on [5 3 8 1 9 2 7 4 6]
  Lomuto, pivot = last = 6
    swap a[0] a[0] -> [5 3 8 1 9 2 7 4 6]
    swap a[1] a[1] -> [5 3 8 1 9 2 7 4 6]
    swap a[2] a[3] -> [5 3 1 8 9 2 7 4 6]
    swap a[3] a[5] -> [5 3 1 2 9 8 7 4 6]
    swap a[4] a[7] -> [5 3 1 2 4 8 7 9 6]
    swap a[5] a[8] -> [5 3 1 2 4 6 7 9 8]  (pivot placed)
  pivot at index 5: left [5 3 1 2 4] | 6 | right [7 9 8]  comparisons 8, swaps 6
  Hoare, pivot = first = 5
    swap a[0] a[7] -> [4 3 8 1 9 2 7 5 6]
    swap a[2] a[5] -> [4 3 2 1 9 8 7 5 6]
  split after index 3: left [4 3 2 1] | right [9 8 7 5 6]  comparisons 11, swaps 2

Section 2: average per partition, 1000 random permutations of n = 1000
  scheme  comparisons    swaps   theory
  Lomuto        999.0    483.5   comparisons n-1 = 999, swaps (n-1)/2+1 = 500.5
  Hoare        1002.0    165.8   comparisons n+O(1), swaps (n-2)/6 = 166.3
  Hoare swaps / Lomuto swaps = 0.343; partition property held: Lomuto 1000/1000, Hoare 1000/1000
  (Lomuto swaps = the pivot's rank, uniform on 1..n; a 1000-trial mean has standard error 9.1)

Section 3: 1000 equal keys
  Lomuto: split 999 | pivot | 0   comparisons 999, swaps 1000
  Hoare:  split 500 | 500          comparisons 1002, swaps 500
  Lomuto sends every equal key left (the next range is n-1 long: quadratic overall);
  Hoare stops on equal keys from both ends and swaps its way to the middle.
```

**Rust:**

```rust
// Pcg: port of Go's math/rand/v2 PCG-DXSM and IntN -- same seed, same permutations.
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
    fn int_n(&mut self, n: u64) -> u64 {
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

#[derive(Default)]
struct Counts {
    cmps: usize,
    swaps: usize,
}

/// Go's %v of a slice: [1 2 3]
fn go_list(a: &[i64]) -> String {
    let parts: Vec<String> = a.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

/// Lomuto around the LAST element. j scans left to right; s is the store index:
/// a[lo..s] holds the elements <= pivot found so far. Every such element is
/// swapped into place -- even when s == j (a self-swap; CLRS does it too, and
/// we count it). CLRS writes i = lo - 1 and pre-increments; the store index
/// s = i + 1 says the same thing without ever going below lo, so usize is fine.
/// Returns the pivot's final index p: a[lo..p] <= a[p] < a[p+1..=hi].
fn lomuto(a: &mut [i64], lo: usize, hi: usize, c: &mut Counts, trace: bool) -> usize {
    let pivot = a[hi];
    let mut s = lo;
    for j in lo..hi {
        c.cmps += 1;
        if a[j] <= pivot {
            a.swap(s, j);
            c.swaps += 1;
            if trace {
                println!("    swap a[{s}] a[{j}] -> {}", go_list(a));
            }
            s += 1;
        }
    }
    a.swap(s, hi); // pivot to its final place
    c.swaps += 1;
    if trace {
        println!("    swap a[{s}] a[{hi}] -> {}  (pivot placed)", go_list(a));
    }
    s
}

/// CLRS HOARE-PARTITION, restructured for usize. CLRS starts i = lo - 1 and
/// j = hi + 1 and uses "repeat j-- until a[j] <= pivot": with lo = 0 that is
/// 0usize - 1, an overflow panic in a debug build. Here i and j hold the NEXT
/// position each side will test (CLRS's i + 1 and j - 1), so each scan tests
/// first and moves after, and both step past a swapped pair explicitly.
/// It tests the same positions in the same order, so the counts match Go's.
/// j never goes below lo: the first scan stops at a[lo] (the pivot itself),
/// later scans stop at or before a slot an earlier swap filled with a small key.
/// Returns j: a[lo..=j] <= a[j+1..=hi]. The pivot is NOT necessarily at j.
fn hoare(a: &mut [i64], lo: usize, hi: usize, c: &mut Counts, trace: bool) -> usize {
    let pivot = a[lo];
    let (mut i, mut j) = (lo, hi);
    loop {
        loop {
            // CLRS: repeat j-- until a[j] <= pivot
            c.cmps += 1;
            if a[j] <= pivot {
                break;
            }
            j -= 1;
        }
        loop {
            // CLRS: repeat i++ until a[i] >= pivot
            c.cmps += 1;
            if a[i] >= pivot {
                break;
            }
            i += 1;
        }
        if i >= j {
            return j;
        }
        a.swap(i, j);
        c.swaps += 1;
        if trace {
            println!("    swap a[{i}] a[{j}] -> {}", go_list(a));
        }
        i += 1; // safe: i < j <= hi
        j -= 1; // safe: j > i >= lo
    }
}

fn permutation(r: &mut Pcg, n: usize) -> Vec<i64> {
    // Fisher-Yates over IntN, in the same order as the Go program
    let mut p: Vec<i64> = (0..n as i64).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i as u64 + 1) as usize;
        p.swap(i, j);
    }
    p
}

fn main() {
    let input = [5, 3, 8, 1, 9, 2, 7, 4, 6];
    println!("Section 1: trace on {}", go_list(&input));
    let mut a = input.to_vec();
    let mut lc = Counts::default();
    println!("  Lomuto, pivot = last = {}", a[a.len() - 1]);
    let hi = a.len() - 1;
    let p = lomuto(&mut a, 0, hi, &mut lc, true);
    println!(
        "  pivot at index {p}: left {} | {} | right {}  comparisons {}, swaps {}",
        go_list(&a[..p]),
        a[p],
        go_list(&a[p + 1..]),
        lc.cmps,
        lc.swaps
    );
    let mut a = input.to_vec();
    let mut hc = Counts::default();
    println!("  Hoare, pivot = first = {}", a[0]);
    let j = hoare(&mut a, 0, hi, &mut hc, true);
    println!(
        "  split after index {j}: left {} | right {}  comparisons {}, swaps {}",
        go_list(&a[..=j]),
        go_list(&a[j + 1..]),
        hc.cmps,
        hc.swaps
    );

    const N: usize = 1000;
    const TRIALS: usize = 1000;
    let mut r = Pcg::new(1, 2);
    let (mut lt, mut ht) = (Counts::default(), Counts::default());
    let (mut ok_l, mut ok_h) = (0, 0);
    for _ in 0..TRIALS {
        let perm = permutation(&mut r, N);
        let mut a = perm.clone();
        let p = lomuto(&mut a, 0, N - 1, &mut lt, false);
        let left_max = a[..=p].iter().max() == Some(&a[p]);
        let right_min = p == N - 1 || a[p..].iter().min() == Some(&a[p]);
        if left_max && right_min {
            ok_l += 1;
        }
        let mut a = perm;
        let j = hoare(&mut a, 0, N - 1, &mut ht, false);
        if a[..=j].iter().max() <= a[j + 1..].iter().min() {
            ok_h += 1;
        }
    }
    let avg = |x: usize| x as f64 / TRIALS as f64;
    println!();
    println!("Section 2: average per partition, {TRIALS} random permutations of n = {N}");
    println!(
        "  {:<7} {:>11} {:>8}   theory",
        "scheme", "comparisons", "swaps"
    );
    println!(
        "  {:<7} {:>11.1} {:>8.1}   comparisons n-1 = {}, swaps (n-1)/2+1 = {:.1}",
        "Lomuto",
        avg(lt.cmps),
        avg(lt.swaps),
        N - 1,
        (N - 1) as f64 / 2.0 + 1.0
    );
    println!(
        "  {:<7} {:>11.1} {:>8.1}   comparisons n+O(1), swaps (n-2)/6 = {:.1}",
        "Hoare",
        avg(ht.cmps),
        avg(ht.swaps),
        (N - 2) as f64 / 6.0
    );
    println!(
        "  Hoare swaps / Lomuto swaps = {:.3}; partition property held: Lomuto {ok_l}/{TRIALS}, Hoare {ok_h}/{TRIALS}",
        ht.swaps as f64 / lt.swaps as f64
    );
    // Lomuto's swap count is the pivot's rank, uniform on 1..n: variance (n*n-1)/12.
    println!(
        "  (Lomuto swaps = the pivot's rank, uniform on 1..n; a {TRIALS}-trial mean has standard error {:.1})",
        ((N * N - 1) as f64 / 12.0 / TRIALS as f64).sqrt()
    );

    println!();
    println!("Section 3: {N} equal keys");
    let (mut le, mut he) = (Counts::default(), Counts::default());
    let p = lomuto(&mut vec![0; N], 0, N - 1, &mut le, false);
    let j = hoare(&mut vec![0; N], 0, N - 1, &mut he, false);
    println!(
        "  Lomuto: split {p} | pivot | {}   comparisons {}, swaps {}",
        N - 1 - p,
        le.cmps,
        le.swaps
    );
    println!(
        "  Hoare:  split {} | {}          comparisons {}, swaps {}",
        j + 1,
        N - 1 - j,
        he.cmps,
        he.swaps
    );
    println!(
        "  Lomuto sends every equal key left (the next range is n-1 long: quadratic overall);"
    );
    println!("  Hoare stops on equal keys from both ends and swaps its way to the middle.");
}
```

**Rust output:**

```
Section 1: trace on [5 3 8 1 9 2 7 4 6]
  Lomuto, pivot = last = 6
    swap a[0] a[0] -> [5 3 8 1 9 2 7 4 6]
    swap a[1] a[1] -> [5 3 8 1 9 2 7 4 6]
    swap a[2] a[3] -> [5 3 1 8 9 2 7 4 6]
    swap a[3] a[5] -> [5 3 1 2 9 8 7 4 6]
    swap a[4] a[7] -> [5 3 1 2 4 8 7 9 6]
    swap a[5] a[8] -> [5 3 1 2 4 6 7 9 8]  (pivot placed)
  pivot at index 5: left [5 3 1 2 4] | 6 | right [7 9 8]  comparisons 8, swaps 6
  Hoare, pivot = first = 5
    swap a[0] a[7] -> [4 3 8 1 9 2 7 5 6]
    swap a[2] a[5] -> [4 3 2 1 9 8 7 5 6]
  split after index 3: left [4 3 2 1] | right [9 8 7 5 6]  comparisons 11, swaps 2

Section 2: average per partition, 1000 random permutations of n = 1000
  scheme  comparisons    swaps   theory
  Lomuto        999.0    483.5   comparisons n-1 = 999, swaps (n-1)/2+1 = 500.5
  Hoare        1002.0    165.8   comparisons n+O(1), swaps (n-2)/6 = 166.3
  Hoare swaps / Lomuto swaps = 0.343; partition property held: Lomuto 1000/1000, Hoare 1000/1000
  (Lomuto swaps = the pivot's rank, uniform on 1..n; a 1000-trial mean has standard error 9.1)

Section 3: 1000 equal keys
  Lomuto: split 999 | pivot | 0   comparisons 999, swaps 1000
  Hoare:  split 500 | 500          comparisons 1002, swaps 500
  Lomuto sends every equal key left (the next range is n-1 long: quadratic overall);
  Hoare stops on equal keys from both ends and swaps its way to the middle.
```

> **Go vs Rust:** Output identical to Go's, character for character, and also identical in a debug build with overflow checks on. CLRS's Hoare starts at `i = lo − 1, j = hi + 1`; in Rust that is `0usize − 1`, an overflow panic in a debug build. The twin does not switch to `isize`. It **restructures the loop** instead: `i` and `j` hold the next position to test, each scan tests first and steps after, and a swapped pair is stepped past explicitly. It tests the same positions in the same order, so the counts are unchanged. Lomuto needed no change, because both versions already use a store index `s` that starts at `lo`.

**Complexity:** Θ(n) per partition, Θ(1) extra space · Lomuto: exactly n − 1 comparisons and 483.5 swaps on average at n = 1,000 (theory (n−1)/2 + 1 = 500.5, standard error 9.1) · Hoare: 1002.0 comparisons (n + O(1)) and 165.8 swaps (theory (n−2)/6 = 166.3), 0.343 × Lomuto's swaps · on 1,000 equal keys Lomuto splits 999 | 0 and Hoare splits 500 | 500.

---

## 5. Quicksort, and the input that makes it quadratic

`🟢 easy` · *Quicksort / Θ(n²) worst case*

Quicksort with the last element as pivot is fast on random input: Θ(n log n), about 2(n+1)H(n) − 4n comparisons.
On sorted input, where you might expect it to be fastest, the pivot is the maximum of every range. Every split is
then n − 1 | 0, and the sort makes exactly n(n−1)/2 comparisons with recursion depth n − 1. Reversed input fails
the same way. That is two costs, and they have different fixes. Recursing on the smaller side and looping on the
larger bounds the **depth** at lg n whatever the pivots do, but it performs the very same partitions, so the
**time** stays quadratic. The program counts comparisons instead of timing them, so even the quadratic rows run
in about a second.

**Steps:**

1. Write Lomuto quicksort on subslices, with a comparison counter and a max-depth counter.
2. Run random, sorted and reversed input at n = 1,000 … 8,000. Compare with n(n−1)/2 and with the exact random-input
   average, and print the doubling ratio.
3. Switch to smaller-side-first recursion and watch the depth collapse while the comparisons stay the same. Then
   run one sorted row at n = 50,000 with plain recursion.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
)

type stats struct{ cmps, maxDepth int }

// lomuto partitions a around its LAST element and returns the pivot's index.
// n-1 comparisons for n elements.
func lomuto(a []int, st *stats) int {
	hi := len(a) - 1
	pivot, s := a[hi], 0
	for j := 0; j < hi; j++ {
		st.cmps++
		if a[j] <= pivot {
			a[s], a[j] = a[j], a[s]
			s++
		}
	}
	a[s], a[hi] = a[hi], a[s]
	return s
}

// quick is textbook quicksort: partition, then recurse on BOTH sides.
// depth counts nested partitioning calls (the root is depth 1).
func quick(a []int, depth int, st *stats) {
	if len(a) < 2 {
		return
	}
	st.maxDepth = max(st.maxDepth, depth)
	p := lomuto(a, st)
	quick(a[:p], depth+1, st)
	quick(a[p+1:], depth+1, st)
}

// quickSmallFirst recurses on the SMALLER side and loops on the larger one.
// Each recursive call gets at most half the range, so depth <= lg n -- but the
// partitions, and so the comparisons, are exactly the same as quick's.
func quickSmallFirst(a []int, depth int, st *stats) {
	for len(a) > 1 {
		st.maxDepth = max(st.maxDepth, depth)
		p := lomuto(a, st)
		left, right := a[:p], a[p+1:]
		if len(left) < len(right) {
			quickSmallFirst(left, depth+1, st)
			a = right // loop instead of a second call: no new stack frame
		} else {
			quickSmallFirst(right, depth+1, st)
			a = left
		}
	}
}

func run(sort func([]int, int, *stats), a []int) stats {
	var st stats
	sort(a, 1, &st)
	if !slices.IsSorted(a) {
		panic("not sorted")
	}
	return st
}

func permutation(r *rand.Rand, n int) []int { // Fisher-Yates over IntN
	p := make([]int, n)
	for i := range p {
		p[i] = i
	}
	for i := n - 1; i > 0; i-- {
		j := r.IntN(i + 1)
		p[i], p[j] = p[j], p[i]
	}
	return p
}

func shape(n int, reversed bool) []int {
	a := make([]int, n)
	for i := range a {
		a[i] = i
		if reversed {
			a[i] = n - 1 - i
		}
	}
	return a
}

// expected is the exact average comparison count of this quicksort on a random
// permutation: 2(n+1)H(n) - 4n, where H(n) = 1 + 1/2 + ... + 1/n.
func expected(n int) float64 {
	h := 0.0
	for k := 1; k <= n; k++ {
		h += 1 / float64(k)
	}
	return 2*float64(n+1)*h - 4*float64(n)
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	fmt.Println("Lomuto quicksort, last element as pivot: comparisons / max depth")
	fmt.Printf("  %5s %14s %10s %18s %18s %10s %7s %7s\n", "n", "random", "expected",
		"sorted", "reversed", "n(n-1)/2", "x rand", "x sort")
	exact := true
	var prevR, prevS int
	loR, hiR, loS, hiS := 1e9, 0.0, 1e9, 0.0 // range of the doubling ratios
	for _, n := range []int{1000, 2000, 4000, 8000} {
		rs := run(quick, permutation(r, n))
		ss := run(quick, shape(n, false))
		vs := run(quick, shape(n, true))
		formula := n * (n - 1) / 2
		exact = exact && ss.cmps == formula && vs.cmps == formula && ss.maxDepth == n-1 && vs.maxDepth == n-1
		ratios := "      -       -"
		if prevR > 0 {
			xr, xs := float64(rs.cmps)/float64(prevR), float64(ss.cmps)/float64(prevS)
			ratios = fmt.Sprintf("%7.2f %7.2f", xr, xs)
			loR, hiR, loS, hiS = min(loR, xr), max(hiR, xr), min(loS, xs), max(hiS, xs)
		}
		fmt.Printf("  %5d %8d / %3d %10.0f %10d / %5d %10d / %5d %10d %s\n", n, rs.cmps, rs.maxDepth,
			expected(n), ss.cmps, ss.maxDepth, vs.cmps, vs.maxDepth, formula, ratios)
		prevR, prevS = rs.cmps, ss.cmps
	}
	fmt.Println("  sorted and reversed hit n(n-1)/2 comparisons and depth n-1 exactly on every row:", exact)
	fmt.Printf("  doubling n multiplies the sorted cost by %.2f-%.2f (quadratic: 4), the random cost by %.2f-%.2f\n",
		loS, hiS, loR, hiR)

	const n = 8000
	fmt.Println()
	fmt.Printf("smaller side first, n = %d: comparisons / max depth\n", n)
	plain := run(quick, shape(n, false))
	small := run(quickSmallFirst, shape(n, false))
	fmt.Printf("  sorted, both sides recursive: %10d / %5d\n", plain.cmps, plain.maxDepth)
	fmt.Printf("  sorted, smaller side first:   %10d / %5d\n", small.cmps, small.maxDepth)
	perm := permutation(r, n)
	plainR := run(quick, slices.Clone(perm))
	smallR := run(quickSmallFirst, perm)
	fmt.Printf("  random, both sides recursive: %10d / %5d\n", plainR.cmps, plainR.maxDepth)
	fmt.Printf("  random, smaller side first:   %10d / %5d   (lg n = %.2f)\n", smallR.cmps, smallR.maxDepth, math.Log2(n))
	fmt.Printf("  comparisons unchanged: sorted %v, random %v -- only the stack depth changes,\n",
		plain.cmps == small.cmps, plainR.cmps == smallR.cmps)
	fmt.Println("  so the depth is fixed and the quadratic time is not: two different failure modes.")

	const deep = 50000
	st := run(quick, shape(deep, false))
	fmt.Println()
	fmt.Printf("sorted n = %d, both sides recursive: %d comparisons, depth %d\n", deep, st.cmps, st.maxDepth)
	fmt.Println("  Go survives this depth: a goroutine's stack starts small and grows (up to 1 GB on 64-bit).")
}
```

**Output:**

```
Lomuto quicksort, last element as pivot: comparisons / max depth
      n         random   expected             sorted           reversed   n(n-1)/2  x rand  x sort
   1000    10800 /  20      10986     499500 /   999     499500 /   999     499500       -       -
   2000    24499 /  27      24730    1999000 /  1999    1999000 /  1999    1999000    2.27    4.00
   4000    54002 /  27      54989    7998000 /  3999    7998000 /  3999    7998000    2.20    4.00
   8000   117236 /  28     121051   31996000 /  7999   31996000 /  7999   31996000    2.17    4.00
  sorted and reversed hit n(n-1)/2 comparisons and depth n-1 exactly on every row: true
  doubling n multiplies the sorted cost by 4.00-4.00 (quadratic: 4), the random cost by 2.17-2.27

smaller side first, n = 8000: comparisons / max depth
  sorted, both sides recursive:   31996000 /  7999
  sorted, smaller side first:     31996000 /     1
  random, both sides recursive:     129321 /    30
  random, smaller side first:       129321 /     8   (lg n = 12.97)
  comparisons unchanged: sorted true, random true -- only the stack depth changes,
  so the depth is fixed and the quadratic time is not: two different failure modes.

sorted n = 50000, both sides recursive: 1249975000 comparisons, depth 49999
  Go survives this depth: a goroutine's stack starts small and grows (up to 1 GB on 64-bit).
```

**Rust:**

```rust
// Pcg: port of Go's math/rand/v2 PCG-DXSM and IntN -- same seed, same permutations.
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
    fn int_n(&mut self, n: u64) -> u64 {
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

#[derive(Default)]
struct Stats {
    cmps: usize,
    max_depth: usize,
}

/// Partitions a around its LAST element; returns the pivot's index.
/// n-1 comparisons for n elements.
fn lomuto(a: &mut [i64], st: &mut Stats) -> usize {
    let hi = a.len() - 1;
    let (pivot, mut s) = (a[hi], 0);
    for j in 0..hi {
        st.cmps += 1;
        if a[j] <= pivot {
            a.swap(s, j);
            s += 1;
        }
    }
    a.swap(s, hi);
    s
}

/// Textbook quicksort: partition, then recurse on BOTH sides.
/// depth counts nested partitioning calls (the root is depth 1).
/// Working on subslices (split_at_mut) means no lo/hi index arithmetic, so no
/// p - 1 that could underflow a usize when the pivot lands at index 0.
fn quick(a: &mut [i64], depth: usize, st: &mut Stats) {
    if a.len() < 2 {
        return;
    }
    st.max_depth = st.max_depth.max(depth);
    let p = lomuto(a, st);
    let (left, right) = a.split_at_mut(p);
    quick(left, depth + 1, st);
    quick(&mut right[1..], depth + 1, st);
}

/// Recurses on the SMALLER side and loops on the larger one. Each recursive
/// call gets at most half the range, so depth <= lg n -- but the partitions,
/// and so the comparisons, are exactly the same as quick's.
fn quick_small_first(mut a: &mut [i64], depth: usize, st: &mut Stats) {
    while a.len() > 1 {
        st.max_depth = st.max_depth.max(depth);
        let p = lomuto(a, st);
        // `a = right` needs the halves to borrow from the ORIGINAL slice, not
        // from `a` itself: mem::take moves the &mut out of `a` (leaving an
        // empty slice behind), and the halves then live as long as it did.
        let (left, right) = std::mem::take(&mut a).split_at_mut(p);
        let right = &mut right[1..];
        if left.len() < right.len() {
            quick_small_first(left, depth + 1, st);
            a = right; // loop instead of a second call: no new stack frame
        } else {
            quick_small_first(right, depth + 1, st);
            a = left;
        }
    }
}

fn run(sort: fn(&mut [i64], usize, &mut Stats), mut a: Vec<i64>) -> Stats {
    let mut st = Stats::default();
    sort(&mut a, 1, &mut st);
    assert!(a.is_sorted(), "not sorted");
    st
}

fn permutation(r: &mut Pcg, n: usize) -> Vec<i64> {
    // Fisher-Yates over IntN, in the same order as the Go program
    let mut p: Vec<i64> = (0..n as i64).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i as u64 + 1) as usize;
        p.swap(i, j);
    }
    p
}

fn shape(n: usize, reversed: bool) -> Vec<i64> {
    let n = n as i64;
    if reversed {
        (0..n).rev().collect()
    } else {
        (0..n).collect()
    }
}

/// Exact average comparisons of this quicksort on a random permutation:
/// 2(n+1)H(n) - 4n, where H(n) = 1 + 1/2 + ... + 1/n (summed in Go's order).
fn expected(n: usize) -> f64 {
    let h: f64 = (1..=n).map(|k| 1.0 / k as f64).sum();
    2.0 * (n + 1) as f64 * h - 4.0 * n as f64
}

fn main() {
    let mut r = Pcg::new(1, 2);
    println!("Lomuto quicksort, last element as pivot: comparisons / max depth");
    println!(
        "  {:>5} {:>14} {:>10} {:>18} {:>18} {:>10} {:>7} {:>7}",
        "n", "random", "expected", "sorted", "reversed", "n(n-1)/2", "x rand", "x sort"
    );
    let mut exact = true;
    let (mut prev_r, mut prev_s) = (0, 0);
    let (mut lo_r, mut hi_r, mut lo_s, mut hi_s) = (1e9_f64, 0.0_f64, 1e9_f64, 0.0_f64);
    for n in [1000, 2000, 4000, 8000] {
        let rs = run(quick, permutation(&mut r, n));
        let ss = run(quick, shape(n, false));
        let vs = run(quick, shape(n, true));
        let formula = n * (n - 1) / 2;
        exact &= ss.cmps == formula
            && vs.cmps == formula
            && ss.max_depth == n - 1
            && vs.max_depth == n - 1;
        let mut ratios = "      -       -".to_string();
        if prev_r > 0 {
            let xr = rs.cmps as f64 / prev_r as f64;
            let xs = ss.cmps as f64 / prev_s as f64;
            ratios = format!("{xr:>7.2} {xs:>7.2}");
            (lo_r, hi_r) = (lo_r.min(xr), hi_r.max(xr));
            (lo_s, hi_s) = (lo_s.min(xs), hi_s.max(xs));
        }
        println!(
            "  {n:>5} {:>8} / {:>3} {:>10.0} {:>10} / {:>5} {:>10} / {:>5} {formula:>10} {ratios}",
            rs.cmps,
            rs.max_depth,
            expected(n),
            ss.cmps,
            ss.max_depth,
            vs.cmps,
            vs.max_depth
        );
        (prev_r, prev_s) = (rs.cmps, ss.cmps);
    }
    println!(
        "  sorted and reversed hit n(n-1)/2 comparisons and depth n-1 exactly on every row: {exact}"
    );
    println!(
        "  doubling n multiplies the sorted cost by {lo_s:.2}-{hi_s:.2} (quadratic: 4), the random cost by {lo_r:.2}-{hi_r:.2}"
    );

    const N: usize = 8000;
    println!();
    println!("smaller side first, n = {N}: comparisons / max depth");
    let plain = run(quick, shape(N, false));
    let small = run(quick_small_first, shape(N, false));
    println!(
        "  sorted, both sides recursive: {:>10} / {:>5}",
        plain.cmps, plain.max_depth
    );
    println!(
        "  sorted, smaller side first:   {:>10} / {:>5}",
        small.cmps, small.max_depth
    );
    let perm = permutation(&mut r, N);
    let plain_r = run(quick, perm.clone());
    let small_r = run(quick_small_first, perm);
    println!(
        "  random, both sides recursive: {:>10} / {:>5}",
        plain_r.cmps, plain_r.max_depth
    );
    println!(
        "  random, smaller side first:   {:>10} / {:>5}   (lg n = {:.2})",
        small_r.cmps,
        small_r.max_depth,
        (N as f64).log2()
    );
    println!(
        "  comparisons unchanged: sorted {}, random {} -- only the stack depth changes,",
        plain.cmps == small.cmps,
        plain_r.cmps == small_r.cmps
    );
    println!("  so the depth is fixed and the quadratic time is not: two different failure modes.");

    // The deep row. Go's goroutine stack grows on demand; a Rust thread's stack
    // is fixed when the thread starts (the main thread's is 8 MiB on macOS and
    // Linux). So this row runs in a spawned thread with a stack sized for it.
    const DEEP: usize = 50000;
    const STACK: usize = 64 << 20;
    let st = std::thread::Builder::new()
        .stack_size(STACK)
        .spawn(|| run(quick, shape(DEEP, false)))
        .expect("spawn")
        .join()
        .expect("join");
    println!();
    println!(
        "sorted n = {DEEP}, both sides recursive: {} comparisons, depth {}",
        st.cmps, st.max_depth
    );
    println!(
        "  Rust runs this depth in a thread spawned with a {} MiB stack: a Rust stack does not grow.",
        STACK >> 20
    );
}
```

**Rust output:**

```
Lomuto quicksort, last element as pivot: comparisons / max depth
      n         random   expected             sorted           reversed   n(n-1)/2  x rand  x sort
   1000    10800 /  20      10986     499500 /   999     499500 /   999     499500       -       -
   2000    24499 /  27      24730    1999000 /  1999    1999000 /  1999    1999000    2.27    4.00
   4000    54002 /  27      54989    7998000 /  3999    7998000 /  3999    7998000    2.20    4.00
   8000   117236 /  28     121051   31996000 /  7999   31996000 /  7999   31996000    2.17    4.00
  sorted and reversed hit n(n-1)/2 comparisons and depth n-1 exactly on every row: true
  doubling n multiplies the sorted cost by 4.00-4.00 (quadratic: 4), the random cost by 2.17-2.27

smaller side first, n = 8000: comparisons / max depth
  sorted, both sides recursive:   31996000 /  7999
  sorted, smaller side first:     31996000 /     1
  random, both sides recursive:     129321 /    30
  random, smaller side first:       129321 /     8   (lg n = 12.97)
  comparisons unchanged: sorted true, random true -- only the stack depth changes,
  so the depth is fixed and the quadratic time is not: two different failure modes.

sorted n = 50000, both sides recursive: 1249975000 comparisons, depth 49999
  Rust runs this depth in a thread spawned with a 64 MiB stack: a Rust stack does not grow.
```

> **Go vs Rust:** Every count is identical to Go's. Only the last line differs, because the stack works differently: a Rust thread's stack is fixed when the thread starts (8 MiB for the main thread), so the depth-49,999 row runs in a thread spawned with `stack_size(64 MiB)`. In a separate run (not shown) a recursive call measured 80 bytes per frame with `-O` and 160 in a debug build. So the 50,000 frames need about 4 MB, or about 8 MB in debug, which is right at the main thread's limit. Subslices plus `split_at_mut` avoid the `p − 1` usize underflow of index-based code. Loop-on-the-larger-side needs `std::mem::take(&mut a)` so the halves can outlive the reassignment of `a`.

**Complexity:** sorted and reversed: exactly n(n−1)/2 comparisons and depth n − 1 (×4.00 per doubling) · random: Θ(n log n), within 4% of 2(n+1)H(n) − 4n here (×2.17–2.27 per doubling), depth 20–30 · smaller side first: the same comparisons, depth 1 on sorted input and 8 on random at n = 8,000 (≤ lg n) · plain recursion at depth 49,999 runs fine in Go, whose goroutine stacks grow up to 1 GB (`maxstacksize` in `runtime/proc.go`).

---

## 6. Heapsort in place

`🟢 easy` · *Heapsort / Θ(n log n), O(1) space*

Heapsort works in two phases. First it builds a max-heap in place, bottom-up: each node is sifted down from the
last parent to the root, which costs Θ(n), at most 2n comparisons. Then it swaps the root, the maximum, to the end
of the slice and sifts the new root down, n − 1 times. It is the only classic sort with **both** a Θ(n log n)
worst case and O(1) extra space, which is why every introsort falls back to it. It pays for that in two ways. It
is not adaptive: sorted input costs as much as random. And it is not stable: swaps carry elements past equal
keys. The heap as a data structure is lesson 20. Here it is only a sort, with the sift-down written as in Go's
own `siftDownOrdered`.

**Steps:**

1. Write `siftDown` (pick the larger child, stop when the root is not less than it) and `heapSort`, counting the
   build and sort-down comparisons separately.
2. Run random, sorted, reversed and all-equal input at n = 1,000, 10,000 and 100,000, then sort 10 tagged records.
3. Sort the same input through `container/heap` (`Init`, then `Pop` n times into a new slice) and compare both
   results with `slices.Sort`.

```go
package main

import (
	"container/heap"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
)

// siftDown restores the max-heap property below root in x[:n]: children of i
// are 2i+1 and 2i+2. As in Go's siftDownOrdered: pick the larger child (one
// comparison, if there are two children), stop when the root is NOT less than
// it (a second comparison), otherwise swap and continue one level down.
func siftDown[E any](x []E, root, n int, less func(a, b E) bool, cmps *int) {
	for {
		child := 2*root + 1
		if child >= n {
			return
		}
		if child+1 < n {
			*cmps++
			if less(x[child], x[child+1]) {
				child++
			}
		}
		*cmps++
		if !less(x[root], x[child]) {
			return // the root is >= its larger child: heap restored
		}
		x[root], x[child] = x[child], x[root]
		root = child
	}
}

// heapSort sorts x in place with O(1) extra space, counting the two phases.
func heapSort[E any](x []E, less func(a, b E) bool) (build, sortdown int) {
	n := len(x)
	for i := n/2 - 1; i >= 0; i-- { // phase 1: heapify bottom-up, Θ(n)
		siftDown(x, i, n, less, &build)
	}
	for end := n - 1; end > 0; end-- { // phase 2: n-1 times move the max to the end
		x[0], x[end] = x[end], x[0]
		siftDown(x, 0, end, less, &sortdown)
	}
	return build, sortdown
}

func lessInt(a, b int) bool { return a < b }

// intHeap is the five-method interface container/heap needs (a MIN-heap).
type intHeap struct {
	a    []int
	cmps int
}

func (h *intHeap) Len() int           { return len(h.a) }
func (h *intHeap) Less(i, j int) bool { h.cmps++; return h.a[i] < h.a[j] }
func (h *intHeap) Swap(i, j int)      { h.a[i], h.a[j] = h.a[j], h.a[i] }
func (h *intHeap) Push(x any)         { h.a = append(h.a, x.(int)) }
func (h *intHeap) Pop() any {
	v := h.a[len(h.a)-1]
	h.a = h.a[:len(h.a)-1]
	return v
}

func permutation(r *rand.Rand, n int) []int { // Fisher-Yates over IntN
	p := make([]int, n)
	for i := range p {
		p[i] = i
	}
	for i := n - 1; i > 0; i-- {
		j := r.IntN(i + 1)
		p[i], p[j] = p[j], p[i]
	}
	return p
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	fmt.Println("heapsort comparisons: build phase, sort-down phase")
	fmt.Printf("  %6s %-9s %8s %7s %9s %9s %9s %8s\n", "n", "shape", "build", "build/n", "sortdown", "total", "2n lg n", "total/n")
	buildOK := true
	ratios, eqPerN := "", "" // sorted/random totals, and all-equal total/n, per n
	for _, n := range []int{1000, 10000, 100000} {
		shapes := []struct {
			name string
			a    []int
		}{{"random", permutation(r, n)}, {"sorted", make([]int, n)}, {"reversed", make([]int, n)}, {"all-equal", make([]int, n)}}
		for i := range n {
			shapes[1].a[i], shapes[2].a[i] = i, n-1-i
		}
		totals := map[string]int{}
		for _, s := range shapes {
			b, d := heapSort(s.a, lessInt)
			totals[s.name] = b + d
			if !slices.IsSorted(s.a) {
				panic("not sorted")
			}
			buildOK = buildOK && b <= 2*n
			fmt.Printf("  %6d %-9s %8d %7.3f %9d %9d %9.0f %8.2f\n", n, s.name, b, float64(b)/float64(n),
				d, b+d, 2*float64(n)*math.Log2(float64(n)), float64(b+d)/float64(n))
		}
		ratios += fmt.Sprintf(" %.3f", float64(totals["sorted"])/float64(totals["random"]))
		eqPerN += fmt.Sprintf(" %.2f", float64(totals["all-equal"])/float64(n))
	}
	fmt.Println("  build <= 2n on every row:", buildOK)
	fmt.Println("  sorted total / random total, per n:" + ratios + "  -- sorted input is no cheaper")
	fmt.Println("  all-equal total / n, per n:" + eqPerN + "  -- every sift stops at once: Θ(n)")

	fmt.Println()
	type rec struct{ key, tag int }
	recs := make([]rec, 10)
	for i := range recs {
		recs[i] = rec{r.IntN(3), i}
	}
	show := func(xs []rec) (s string) {
		for _, x := range xs {
			s += fmt.Sprintf(" %d:%c", x.key, 'a'+x.tag)
		}
		return s
	}
	fmt.Println("10 records key:tag, keys IntN(3)")
	fmt.Println("  input:   " + show(recs))
	heapSort(recs, func(a, b rec) bool { return a.key < b.key })
	stable := true
	for i := 1; i < len(recs); i++ {
		if recs[i-1].key == recs[i].key && recs[i-1].tag > recs[i].tag {
			stable = false
		}
	}
	fmt.Println("  heapsort:" + show(recs))
	fmt.Println("  stable:", stable, "-- the swaps carry elements far past equal keys")

	const n = 100000
	in := permutation(r, n)
	want := slices.Clone(in)
	slices.Sort(want)
	h := &intHeap{a: slices.Clone(in)}
	heap.Init(h) // Floyd's bottom-up build, as in heapSort
	out := make([]int, 0, n)
	for h.Len() > 0 {
		out = append(out, heap.Pop(h).(int)) // min first: ascending, into a NEW slice
	}
	mine := slices.Clone(in)
	b, d := heapSort(mine, lessInt)
	fmt.Println()
	fmt.Printf("standard library route, n = %d random\n", n)
	fmt.Printf("  container/heap Init + %d Pops: equal to slices.Sort: %v, Less calls %d\n", n, slices.Equal(out, want), h.cmps)
	fmt.Printf("  in-place heapSort:                equal to slices.Sort: %v, comparisons %d\n", slices.Equal(mine, want), b+d)
}
```

**Output:**

```
heapsort comparisons: build phase, sort-down phase
       n shape        build build/n  sortdown     total   2n lg n  total/n
    1000 random        1876   1.876     15026     16902     19932    16.90
    1000 sorted        1982   1.982     15601     17583     19932    17.58
    1000 reversed       999   0.999     14966     15965     19932    15.96
    1000 all-equal      999   0.999      1995      2994     19932     2.99
   10000 random       18788   1.879    216612    235400    265754    23.54
   10000 sorted       19982   1.998    224478    244460    265754    24.45
   10000 reversed      9999   1.000    216683    226682    265754    22.67
   10000 all-equal     9999   1.000     19995     29994    265754     3.00
  100000 random      187921   1.879   2831570   3019491   3321928    30.19
  100000 sorted      199978   2.000   2912539   3112517   3321928    31.13
  100000 reversed     99999   1.000   2826641   2926640   3321928    29.27
  100000 all-equal    99999   1.000    199995    299994   3321928     3.00
  build <= 2n on every row: true
  sorted total / random total, per n: 1.040 1.038 1.031  -- sorted input is no cheaper
  all-equal total / n, per n: 2.99 3.00 3.00  -- every sift stops at once: Θ(n)

10 records key:tag, keys IntN(3)
  input:    1:a 1:b 2:c 2:d 0:e 2:f 0:g 0:h 1:i 1:j
  heapsort: 0:e 0:g 0:h 1:i 1:j 1:b 1:a 2:f 2:c 2:d
  stable: false -- the swaps carry elements far past equal keys

standard library route, n = 100000 random
  container/heap Init + 100000 Pops: equal to slices.Sort: true, Less calls 3019394
  in-place heapSort:                equal to slices.Sort: true, comparisons 3019829
```

**Rust:**

```rust
use std::cmp::Ordering;
use std::collections::BinaryHeap;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Pcg: port of Go's math/rand/v2 PCG-DXSM and IntN -- same seed, same inputs.
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
    fn int_n(&mut self, n: u64) -> u64 {
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

/// Restores the max-heap property below `root` in x[..n]: children of i are
/// 2i+1 and 2i+2. As in Go's siftDownOrdered: pick the larger child (one
/// comparison, if there are two children), stop when the root is NOT less than
/// it (a second comparison), otherwise swap and continue one level down.
fn sift_down<T>(x: &mut [T], mut root: usize, n: usize, less: &impl Fn(&T, &T) -> bool) -> usize {
    let mut cmps = 0;
    loop {
        let mut child = 2 * root + 1;
        if child >= n {
            return cmps;
        }
        if child + 1 < n {
            cmps += 1;
            if less(&x[child], &x[child + 1]) {
                child += 1;
            }
        }
        cmps += 1;
        if !less(&x[root], &x[child]) {
            return cmps; // the root is >= its larger child: heap restored
        }
        x.swap(root, child);
        root = child;
    }
}

/// Sorts x in place with O(1) extra space; returns (build, sortdown) comparisons.
fn heap_sort<T>(x: &mut [T], less: impl Fn(&T, &T) -> bool) -> (usize, usize) {
    let n = x.len();
    let (mut build, mut sortdown) = (0, 0);
    // phase 1: heapify bottom-up, Θ(n). `(0..n / 2).rev()` is Go's
    // `for i := n/2 - 1; i >= 0; i--` without an i that has to reach -1.
    for i in (0..n / 2).rev() {
        build += sift_down(x, i, n, &less);
    }
    // phase 2: n-1 times move the max to the end
    for end in (1..n).rev() {
        x.swap(0, end);
        sortdown += sift_down(x, 0, end, &less);
    }
    (build, sortdown)
}

// A counting newtype for BinaryHeap, whose sift code calls <=, < and >= on T:
// every one of those goes through partial_cmp -> cmp here and counts once.
static CMPS: AtomicUsize = AtomicUsize::new(0);

struct Counted(i64);

impl Ord for Counted {
    fn cmp(&self, other: &Self) -> Ordering {
        CMPS.fetch_add(1, Relaxed);
        self.0.cmp(&other.0)
    }
}
impl PartialOrd for Counted {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}
impl PartialEq for Counted {
    fn eq(&self, other: &Self) -> bool {
        self.0 == other.0
    }
}
impl Eq for Counted {}

fn permutation(r: &mut Pcg, n: usize) -> Vec<i64> {
    // Fisher-Yates over IntN, in the same order as the Go program
    let mut p: Vec<i64> = (0..n as i64).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i as u64 + 1) as usize;
        p.swap(i, j);
    }
    p
}

fn main() {
    let mut r = Pcg::new(1, 2);
    println!("heapsort comparisons: build phase, sort-down phase");
    println!(
        "  {:>6} {:<9} {:>8} {:>7} {:>9} {:>9} {:>9} {:>8}",
        "n", "shape", "build", "build/n", "sortdown", "total", "2n lg n", "total/n"
    );
    let mut build_ok = true;
    let (mut ratios, mut eq_per_n) = (String::new(), String::new());
    for n in [1000usize, 10000, 100000] {
        let ni = n as i64;
        let shapes: [(&str, Vec<i64>); 4] = [
            ("random", permutation(&mut r, n)),
            ("sorted", (0..ni).collect()),
            ("reversed", (0..ni).rev().collect()),
            ("all-equal", vec![0; n]),
        ];
        let mut totals = [0usize; 4];
        for (k, (name, mut a)) in shapes.into_iter().enumerate() {
            let (b, d) = heap_sort(&mut a, |x, y| x < y);
            assert!(a.is_sorted(), "not sorted");
            totals[k] = b + d;
            build_ok &= b <= 2 * n;
            println!(
                "  {n:>6} {name:<9} {b:>8} {:>7.3} {d:>9} {:>9} {:>9.0} {:>8.2}",
                b as f64 / n as f64,
                b + d,
                2.0 * n as f64 * (n as f64).log2(),
                (b + d) as f64 / n as f64
            );
        }
        ratios += &format!(" {:.3}", totals[1] as f64 / totals[0] as f64);
        eq_per_n += &format!(" {:.2}", totals[3] as f64 / n as f64);
    }
    println!("  build <= 2n on every row: {build_ok}");
    println!("  sorted total / random total, per n:{ratios}  -- sorted input is no cheaper");
    println!("  all-equal total / n, per n:{eq_per_n}  -- every sift stops at once: Θ(n)");

    println!();
    #[derive(Clone, Copy)]
    struct Rec {
        key: u64,
        tag: u8,
    }
    let mut recs: Vec<Rec> = (0..10u8)
        .map(|tag| Rec {
            key: r.int_n(3),
            tag,
        })
        .collect();
    let show = |xs: &[Rec]| -> String {
        xs.iter()
            .map(|x| format!(" {}:{}", x.key, char::from(b'a' + x.tag)))
            .collect()
    };
    println!("10 records key:tag, keys IntN(3)");
    println!("  input:   {}", show(&recs));
    heap_sort(&mut recs, |a, b| a.key < b.key);
    let stable = recs
        .windows(2)
        .all(|w| w[0].key != w[1].key || w[0].tag < w[1].tag);
    println!("  heapsort:{}", show(&recs));
    println!("  stable: {stable} -- the swaps carry elements far past equal keys");

    const N: usize = 100000;
    let input = permutation(&mut r, N);
    let mut want = input.clone();
    want.sort_unstable();
    // BinaryHeap::from(vec) heapifies the Vec's own buffer bottom-up (Floyd);
    // into_sorted_vec then swaps the max to the end and sifts, in that same
    // buffer: an in-place heapsort, max-heap, ascending output. No interface to
    // implement (T: Ord is enough) and no second Vec.
    let heap = BinaryHeap::from(input.iter().map(|&v| Counted(v)).collect::<Vec<_>>());
    let c0 = CMPS.load(Relaxed);
    let sorted = heap.into_sorted_vec();
    let heap_cmps = CMPS.load(Relaxed);
    let heap_equal = sorted.iter().map(|c| c.0).eq(want.iter().copied());
    let mut mine = input;
    let (b, d) = heap_sort(&mut mine, |x, y| x < y);
    println!();
    println!("standard library route, n = {N} random");
    println!(
        "  BinaryHeap::from + into_sorted_vec: equal to sort_unstable: {heap_equal}, comparisons {heap_cmps} (build {c0})"
    );
    println!(
        "  in-place heap_sort:                 equal to sort_unstable: {}, comparisons {} (build {b})",
        mine == want,
        b + d
    );
    println!(
        "  same comparison count as heap_sort, phase by phase: {}",
        heap_cmps == b + d && c0 == b
    );
}
```

**Rust output:**

```
heapsort comparisons: build phase, sort-down phase
       n shape        build build/n  sortdown     total   2n lg n  total/n
    1000 random        1876   1.876     15026     16902     19932    16.90
    1000 sorted        1982   1.982     15601     17583     19932    17.58
    1000 reversed       999   0.999     14966     15965     19932    15.96
    1000 all-equal      999   0.999      1995      2994     19932     2.99
   10000 random       18788   1.879    216612    235400    265754    23.54
   10000 sorted       19982   1.998    224478    244460    265754    24.45
   10000 reversed      9999   1.000    216683    226682    265754    22.67
   10000 all-equal     9999   1.000     19995     29994    265754     3.00
  100000 random      187921   1.879   2831570   3019491   3321928    30.19
  100000 sorted      199978   2.000   2912539   3112517   3321928    31.13
  100000 reversed     99999   1.000   2826641   2926640   3321928    29.27
  100000 all-equal    99999   1.000    199995    299994   3321928     3.00
  build <= 2n on every row: true
  sorted total / random total, per n: 1.040 1.038 1.031  -- sorted input is no cheaper
  all-equal total / n, per n: 2.99 3.00 3.00  -- every sift stops at once: Θ(n)

10 records key:tag, keys IntN(3)
  input:    1:a 1:b 2:c 2:d 0:e 2:f 0:g 0:h 1:i 1:j
  heapsort: 0:e 0:g 0:h 1:i 1:j 1:b 1:a 2:f 2:c 2:d
  stable: false -- the swaps carry elements far past equal keys

standard library route, n = 100000 random
  BinaryHeap::from + into_sorted_vec: equal to sort_unstable: true, comparisons 3019829 (build 187934)
  in-place heap_sort:                 equal to sort_unstable: true, comparisons 3019829 (build 187934)
  same comparison count as heap_sort, phase by phase: true
```

> **Go vs Rust:** Everything except the last section is identical to Go's, character for character. In the last section, `BinaryHeap::from(v).into_sorted_vec()` turns out to be **this same heapsort**, done in the heap's own buffer. It made exactly 3,019,829 comparisons, 187,934 of them in the build, phase for phase the same as the hand-written one. Its `sift_down_range` in `alloc/src/collections/binary_heap/mod.rs` (read in the source) picks the larger child and stops on `>=` just as `siftDownOrdered` does, and on distinct keys that gives the same choices. Go's `container/heap` needs a five-method interface, builds a min-heap, and pops into a second slice: 3,019,394 `Less` calls on the same input, a different count because the heap is ordered the other way.

**Complexity:** Θ(n log n) worst case · build Θ(n): ≤ 2n comparisons (1.88n random, 2.00n sorted, 1.00n reversed) · total 0.85–0.91 × 2n lg n on random input · sorted costs 3–4% *more* than random (not adaptive) · all-equal 3.00n (Θ(n)) · O(1) extra space · not stable. `container/heap` gives the same sorted result but needs a five-method interface, a min-heap and a second slice.

---

> Next tier: [🟡 medium](2-medium.md) — natural runs, six pivot rules, a third partition for duplicates,
> quickselect, and what `slices.Sort` actually runs.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
