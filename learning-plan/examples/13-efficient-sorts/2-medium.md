# Step 13 — Efficient Sorts: Divide & Conquer · 🟡 Medium

Examples **7–11**: bottom-up and natural merge sort, six pivot rules on six input shapes, the 3-way
partition, quickselect, and the standard library's sorts opened up and counted.

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

> ← Back to [🟢 easy](1-easy.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🔴 hard](3-hard.md)

---

## 7. Bottom-up and natural merge sort

`🟡 medium` · *Merge sort / iterative and adaptive*

Merge sort does not need recursion. Bottom-up merge sort merges runs of width 1, then 2, then 4, and after
every pass it swaps the roles of the array and the buffer, so the data is never copied back: about half the
element writes of the top-down version for the same Θ(n log n) comparisons. Once the widths are gone you can
start from the runs the input *already has*. A **natural** merge sort scans for them first (ascending, or
strictly descending and reversed in place, as Rust's `find_existing_run` does), so sorted and reversed input
cost exactly n − 1 comparisons and no merge at all, and k runs need only ⌈lg k⌉ passes. It is still Θ(n log n)
in the worst case, with Θ(n) extra space. This is the skeleton of every real stable sort; Rust's driftsort is
one, and Go's `SortStableFunc` is a different design, which the counts below make visible.

**Steps:**

1. Write one `merge(dst, src, lo, mid, hi)` that counts comparisons and element writes, and build top-down,
   bottom-up (ping-pong buffers) and natural (run scan, then bottom-up over the runs) on it.
2. Run all three on five 100,000-element shapes: random, sorted, reversed, sorted with 10 random swaps, and 16
   sorted runs.
3. Compare comparisons, moves and passes, then count `slices.SortStableFunc` on the same shapes.

```go
package main

import (
	"cmp"
	"fmt"
	"math/rand/v2"
	"slices"
	"strconv"
)

// Three merge sorts. All are Θ(n log n) on random input; they differ in how
// much they write, and in what they do with order the input already has.
//
//	top-down   recurse to the middle, merge into a buffer, copy back (example 2)
//	bottom-up  no recursion: merge widths 1, 2, 4, ... and swap the roles of
//	           the two arrays after every pass, so nothing is copied back
//	natural    bottom-up, but start from the runs the input already has

// counter records the two costs of a merge sort:
//
//	cmps   calls to less -- the comparator
//	moves  element writes into either array (a swap writes two)
type counter struct {
	cmps, moves int
}

func (c *counter) less(a, b int) bool {
	c.cmps++
	return a < b
}

// merge merges src[lo:mid] and src[mid:hi] into dst[lo:hi]. Take from the right
// only when it is STRICTLY smaller, so ties go left and the sort is stable.
// When one run is used up the rest of the other is copied: no comparisons.
func (c *counter) merge(dst, src []int, lo, mid, hi int) {
	i, j, k := lo, mid, lo
	for i < mid && j < hi {
		if c.less(src[j], src[i]) {
			dst[k] = src[j]
			j++
		} else {
			dst[k] = src[i]
			i++
		}
		k++
	}
	k += copy(dst[k:], src[i:mid])
	copy(dst[k:hi], src[j:hi])
	c.moves += hi - lo // every element of the range is written once
}

// topDown: the recursive merge sort, one buffer allocated once. passes is the
// number of merge LEVELS (the deepest recursion level that merges).
func topDown(xs []int) (c counter, passes int) {
	buf := make([]int, len(xs))
	var rec func(lo, hi, depth int)
	rec = func(lo, hi, depth int) {
		if hi-lo < 2 {
			return
		}
		passes = max(passes, depth)
		mid := lo + (hi-lo)/2
		rec(lo, mid, depth+1)
		rec(mid, hi, depth+1)
		c.merge(buf, xs, lo, mid, hi)
		copy(xs[lo:hi], buf[lo:hi]) // the copy back: the price of recursion
		c.moves += hi - lo
	}
	rec(0, len(xs), 1)
	return c, passes
}

// bottomUp: merge pairs of width-w runs from src into dst, then swap the two
// slices -- the output of one pass is the input of the next. A lone run at the
// end of a pass is merged with an empty one, i.e. copied (0 comparisons).
func bottomUp(xs []int) (c counter, passes int) {
	n := len(xs)
	src, dst := xs, make([]int, n)
	for width := 1; width < n; width *= 2 {
		for lo := 0; lo < n; lo += 2 * width {
			c.merge(dst, src, lo, min(lo+width, n), min(lo+2*width, n))
		}
		src, dst = dst, src // ping-pong: no copy back
		passes++
	}
	if passes%2 == 1 { // the sorted data ended up in the buffer
		copy(xs, src)
		c.moves += n
	}
	return c, passes
}

// natural: scan for the runs that are already there, then merge runs pairwise,
// pass after pass, exactly like bottom-up. The scan follows Rust's
// find_existing_run: a run is non-descending, or STRICTLY descending (then it is
// reversed in place). Strict, because reversing equal keys would break
// stability. Each adjacent pair is compared once, so the scan costs n-1.
func natural(xs []int) (c counter, runs, passes int) {
	n := len(xs)
	bounds := []int{0} // run i is xs[bounds[i]:bounds[i+1]]
	for i := 0; i < n; {
		j := i + 1
		if j < n && c.less(xs[j], xs[i]) { // strictly descending
			for j++; j < n && c.less(xs[j], xs[j-1]); j++ {
			}
			for a, b := i, j-1; a < b; a, b = a+1, b-1 {
				xs[a], xs[b] = xs[b], xs[a]
				c.moves += 2
			}
		} else if j < n { // non-descending
			for j++; j < n && !c.less(xs[j], xs[j-1]); j++ {
			}
		}
		bounds = append(bounds, j)
		i = j
	}
	runs = len(bounds) - 1
	if runs == 1 {
		return c, runs, 0 // already sorted: no buffer, no merge
	}
	src, dst := xs, make([]int, n)
	for len(bounds) > 2 {
		next := []int{0}
		for r := 0; r+1 < len(bounds); r += 2 {
			lo, mid, hi := bounds[r], bounds[r+1], bounds[r+1]
			if r+2 < len(bounds) {
				hi = bounds[r+2]
			}
			c.merge(dst, src, lo, mid, hi) // lone last run: mid == hi, a copy
			next = append(next, hi)
		}
		bounds = next
		src, dst = dst, src
		passes++
	}
	if passes%2 == 1 {
		copy(xs, src)
		c.moves += n
	}
	return c, runs, passes
}

// commas formats 1234567 as "1,234,567".
func commas(v int) string {
	s := strconv.Itoa(v)
	out := ""
	for len(s) > 3 {
		out = "," + s[len(s)-3:] + out
		s = s[:len(s)-3]
	}
	return s + out
}

func main() {
	const n = 100_000 // not a power of two: top-down and bottom-up pair runs differently
	r := rand.New(rand.NewPCG(1, 2))

	// Fisher-Yates over IntN, so the Rust twin sees the same permutations.
	perm := func() []int {
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
	random := perm()
	sorted := make([]int, n)
	reversed := make([]int, n)
	for i := range n {
		sorted[i] = i
		reversed[i] = n - 1 - i
	}
	swapped := slices.Clone(sorted)
	for range 10 {
		i := r.IntN(n)
		j := r.IntN(n)
		swapped[i], swapped[j] = swapped[j], swapped[i]
	}
	runs16 := perm() // a permutation cut into 16 blocks, each sorted
	for b := range 16 {
		slices.Sort(runs16[b*n/16 : (b+1)*n/16])
	}

	shapes := []struct {
		name string
		xs   []int
	}{
		{"random", random}, {"sorted", sorted}, {"reversed", reversed},
		{"sorted+10 swaps", swapped}, {"16 sorted runs", runs16},
	}

	fmt.Printf("Section 1: three merge sorts, n = %s\n\n", commas(n))
	fmt.Printf("%-16s %-10s %12s %12s %7s %8s\n", "shape", "sort", "comparisons", "moves", "passes", "runs")
	allSorted := true
	natCmps, natRuns, natPasses := map[string]int{}, map[string]int{}, map[string]int{}
	var tdRandom, buRandom counter
	tdRandomP := 0
	for _, s := range shapes {
		a, b, c := slices.Clone(s.xs), slices.Clone(s.xs), slices.Clone(s.xs)
		td, tdP := topDown(a)
		bu, buP := bottomUp(b)
		nat, runs, natP := natural(c)
		allSorted = allSorted && slices.IsSorted(a) && slices.IsSorted(b) && slices.IsSorted(c)
		natCmps[s.name], natRuns[s.name], natPasses[s.name] = nat.cmps, runs, natP
		if s.name == "random" {
			tdRandom, buRandom, tdRandomP = td, bu, tdP
		}
		fmt.Printf("%-16s %-10s %12s %12s %7d %8s\n", s.name, "top-down", commas(td.cmps), commas(td.moves), tdP, "-")
		fmt.Printf("%-16s %-10s %12s %12s %7d %8s\n", "", "bottom-up", commas(bu.cmps), commas(bu.moves), buP, "-")
		fmt.Printf("%-16s %-10s %12s %12s %7d %8s\n", "", "natural", commas(nat.cmps), commas(nat.moves), natP, commas(runs))
	}
	fmt.Println()
	fmt.Println("every output sorted:", allSorted)
	fmt.Printf("n-1 = %s; natural on sorted = %s, on reversed = %s (scan only, 0 merges)\n",
		commas(n-1), commas(natCmps["sorted"]), commas(natCmps["reversed"]))
	d, word := buRandom.cmps-tdRandom.cmps, "more"
	if d < 0 {
		d, word = -d, "fewer"
	}
	fmt.Printf("random: bottom-up makes %s %s comparisons than top-down (n is not a power of\n",
		commas(d), word)
	fmt.Printf("  two, so the pairings differ) and %.2fx its moves -- ping-pong drops the copy back\n",
		float64(buRandom.moves)/float64(tdRandom.moves))
	fmt.Printf("random: natural finds %s runs (avg length %.2f): the scan saves %d pass(es)\n",
		commas(natRuns["random"]), float64(n)/float64(natRuns["random"]), tdRandomP-natPasses["random"])
	fmt.Printf("  and costs %s more comparisons than top-down\n", commas(natCmps["random"]-tdRandom.cmps))
	fmt.Printf("sorted+10 swaps: %d runs -> %d passes, %s comparisons = %.1fx n: a plain merge\n",
		natRuns["sorted+10 swaps"], natPasses["sorted+10 swaps"], commas(natCmps["sorted+10 swaps"]),
		float64(natCmps["sorted+10 swaps"])/n)
	fmt.Println("  walks an ordered stretch one comparison per element, pass after pass")

	// Section 2: the standard library's stable sort, counted through its comparator.
	// Go's slices.SortStableFunc insertion-sorts blocks of 20 and merges them in
	// place with SymMerge (src/slices/zsortanyfunc.go, stableCmpFunc): it never
	// looks for runs, so sorted input is not Θ(n) for it.
	fmt.Println()
	fmt.Println("Section 2: slices.SortStableFunc (blocks of 20 + SymMerge), same shapes")
	fmt.Println()
	fmt.Printf("%-16s %12s %12s %8s\n", "shape", "comparisons", "natural", "ratio")
	std := map[string]int{}
	for _, s := range shapes {
		xs := slices.Clone(s.xs)
		cmps := 0
		slices.SortStableFunc(xs, func(a, b int) int {
			cmps++
			return cmp.Compare(a, b)
		})
		std[s.name] = cmps
		fmt.Printf("%-16s %12s %12s %8.2f\n", s.name, commas(cmps), commas(natCmps[s.name]),
			float64(cmps)/float64(natCmps[s.name]))
	}
	fmt.Println()
	fmt.Printf("it never looks for runs: reversed costs %.1fx sorted, where natural pays n-1 for both\n",
		float64(std["reversed"])/float64(std["sorted"]))
	if std["sorted+10 swaps"] < natCmps["sorted+10 swaps"] {
		fmt.Printf("but on sorted+10 swaps it wins (%s vs %s): SymMerge binary-searches its split\n",
			commas(std["sorted+10 swaps"]), commas(natCmps["sorted+10 swaps"]))
		fmt.Println("points, so an ordered stretch costs O(log n), not a linear merge")
	}
}
```

**Output:**

```
Section 1: three merge sorts, n = 100,000

shape            sort        comparisons        moves  passes     runs
random           top-down      1,536,358    3,337,856      17        -
                 bottom-up     1,566,509    1,800,000      17        -
                 natural       1,610,328    1,644,944      16   41,380
sorted           top-down        815,024    3,337,856      17        -
                 bottom-up       877,968    1,800,000      17        -
                 natural          99,999            0       0        1
reversed         top-down        853,904    3,337,856      17        -
                 bottom-up       815,024    1,800,000      17        -
                 natural          99,999      100,000       0        1
sorted+10 swaps  top-down        981,341    3,337,856      17        -
                 bottom-up     1,042,676    1,800,000      17        -
                 natural         526,932      600,000       5       21
16 sorted runs   top-down      1,014,981    3,337,856      17        -
                 bottom-up     1,119,054    1,800,000      17        -
                 natural         499,956      400,000       4       16

every output sorted: true
n-1 = 99,999; natural on sorted = 99,999, on reversed = 99,999 (scan only, 0 merges)
random: bottom-up makes 30,151 more comparisons than top-down (n is not a power of
  two, so the pairings differ) and 0.54x its moves -- ping-pong drops the copy back
random: natural finds 41,380 runs (avg length 2.42): the scan saves 1 pass(es)
  and costs 73,970 more comparisons than top-down
sorted+10 swaps: 21 runs -> 5 passes, 526,932 comparisons = 5.3x n: a plain merge
  walks an ordered stretch one comparison per element, pass after pass

Section 2: slices.SortStableFunc (blocks of 20 + SymMerge), same shapes

shape             comparisons      natural    ratio
random              2,098,212    1,610,328     1.30
sorted                120,210       99,999     1.20
reversed              980,261       99,999     9.80
sorted+10 swaps       122,531      526,932     0.23
16 sorted runs        630,987      499,956     1.26

it never looks for runs: reversed costs 8.2x sorted, where natural pays n-1 for both
but on sorted+10 swaps it wins (122,531 vs 526,932): SymMerge binary-searches its split
points, so an ordered stretch costs O(log n), not a linear merge
```

**Rust:**

```rust
use std::collections::HashMap;

// Three merge sorts. All are Θ(n log n) on random input; they differ in how
// much they write, and in what they do with order the input already has.
//
//   top-down   recurse to the middle, merge into a buffer, copy back (example 2)
//   bottom-up  no recursion: merge widths 1, 2, 4, ... and swap the roles of
//              the two slices after every pass, so nothing is copied back
//   natural    bottom-up, but start from the runs the input already has

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so this
// program sorts exactly the permutations the Go version sorts.
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
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
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

// Counter records the two costs of a merge sort:
//   cmps   calls to less -- the comparator
//   moves  element writes into either slice (a swap writes two)
#[derive(Default, Clone, Copy)]
struct Counter {
    cmps: usize,
    moves: usize,
}

impl Counter {
    fn less(&mut self, a: i64, b: i64) -> bool {
        self.cmps += 1;
        a < b
    }

    // merge `left` and `right` into `dst` (dst.len() == left.len() + right.len()).
    // Take from the right only when it is STRICTLY smaller: ties go left, stable.
    // When one run is used up the rest of the other is copied: no comparisons.
    fn merge(&mut self, dst: &mut [i64], left: &[i64], right: &[i64]) {
        let (mut i, mut j, mut k) = (0, 0, 0);
        while i < left.len() && j < right.len() {
            if self.less(right[j], left[i]) {
                dst[k] = right[j];
                j += 1;
            } else {
                dst[k] = left[i];
                i += 1;
            }
            k += 1;
        }
        let rest = left.len() - i;
        dst[k..k + rest].copy_from_slice(&left[i..]);
        dst[k + rest..].copy_from_slice(&right[j..]);
        self.moves += dst.len(); // every element of the range is written once
    }
}

// top_down: the recursive merge sort with one buffer allocated by the caller.
// split_at_mut hands out the two halves of xs AND of buf as disjoint &mut
// slices -- the borrow checker's proof that the recursive calls cannot overlap.
// Returns the number of merge LEVELS below and including this call.
fn top_down(xs: &mut [i64], buf: &mut [i64], c: &mut Counter) -> usize {
    if xs.len() < 2 {
        return 0;
    }
    let mid = xs.len() / 2;
    let levels = {
        let (xl, xr) = xs.split_at_mut(mid);
        let (bl, br) = buf.split_at_mut(mid);
        top_down(xl, bl, c).max(top_down(xr, br, c))
    };
    let (left, right) = xs.split_at(mid);
    c.merge(buf, left, right);
    xs.copy_from_slice(buf); // the copy back: the price of recursion
    c.moves += xs.len();
    levels + 1
}

// bottom_up: merge pairs of width-w runs from src into dst, then swap the two
// &mut slices -- the output of one pass is the input of the next. chunks()
// yields the pairs; a lone last run is merged with an empty one (a copy).
fn bottom_up(xs: &mut [i64], c: &mut Counter) -> usize {
    let n = xs.len();
    let mut buf = vec![0; n];
    let (mut src, mut dst): (&mut [i64], &mut [i64]) = (xs, &mut buf);
    let (mut width, mut passes) = (1, 0);
    while width < n {
        for (d, s) in dst.chunks_mut(2 * width).zip(src.chunks(2 * width)) {
            let m = width.min(s.len());
            c.merge(d, &s[..m], &s[m..]);
        }
        std::mem::swap(&mut src, &mut dst); // ping-pong: no copy back
        passes += 1;
        width *= 2;
    }
    if passes % 2 == 1 {
        // the sorted data ended up in the buffer (src); dst is the caller's xs
        dst.copy_from_slice(src);
        c.moves += n;
    }
    passes
}

// natural: scan for the runs that are already there, then merge runs pairwise,
// pass after pass, exactly like bottom-up. The scan is std's find_existing_run:
// a run is non-descending, or STRICTLY descending (then reversed in place).
// Strict, because reversing equal keys would break stability. Each adjacent
// pair is compared once, so the scan costs n-1. Returns (runs, passes).
fn natural(xs: &mut [i64], c: &mut Counter) -> (usize, usize) {
    let n = xs.len();
    let mut bounds = vec![0]; // run i is xs[bounds[i]..bounds[i+1]]
    let mut i = 0;
    while i < n {
        let mut j = i + 1;
        if j < n && c.less(xs[j], xs[i]) {
            j += 1;
            while j < n && c.less(xs[j], xs[j - 1]) {
                j += 1;
            }
            xs[i..j].reverse(); // len/2 swaps
            c.moves += 2 * ((j - i) / 2);
        } else if j < n {
            j += 1;
            while j < n && !c.less(xs[j], xs[j - 1]) {
                j += 1;
            }
        }
        bounds.push(j);
        i = j;
    }
    let runs = bounds.len() - 1;
    if runs == 1 {
        return (1, 0); // already sorted: no buffer, no merge
    }
    let mut buf = vec![0; n];
    let (mut src, mut dst): (&mut [i64], &mut [i64]) = (xs, &mut buf);
    let mut passes = 0;
    while bounds.len() > 2 {
        let mut next = vec![0];
        for r in (0..bounds.len() - 1).step_by(2) {
            let (lo, mid) = (bounds[r], bounds[r + 1]);
            let hi = bounds.get(r + 2).copied().unwrap_or(mid); // lone run: a copy
            c.merge(&mut dst[lo..hi], &src[lo..mid], &src[mid..hi]);
            next.push(hi);
        }
        bounds = next;
        std::mem::swap(&mut src, &mut dst);
        passes += 1;
    }
    if passes % 2 == 1 {
        dst.copy_from_slice(src);
        c.moves += n;
    }
    (runs, passes)
}

/// commas formats 1234567 as "1,234,567".
fn commas(v: usize) -> String {
    let s = v.to_string();
    let mut out = String::new();
    for (k, ch) in s.chars().enumerate() {
        if k > 0 && (s.len() - k).is_multiple_of(3) {
            out.push(',');
        }
        out.push(ch);
    }
    out
}

fn main() {
    const N: usize = 100_000; // not a power of two: top-down and bottom-up pair runs differently
    let mut r = Pcg::new(1, 2);

    // Fisher-Yates over int_n, in the same order as the Go version.
    let perm = |r: &mut Pcg| -> Vec<i64> {
        let mut p: Vec<i64> = (0..N as i64).collect();
        for i in (1..N).rev() {
            let j = r.int_n(i + 1);
            p.swap(i, j);
        }
        p
    };
    let random = perm(&mut r);
    let sorted: Vec<i64> = (0..N as i64).collect();
    let reversed: Vec<i64> = (0..N as i64).rev().collect();
    let mut swapped = sorted.clone();
    for _ in 0..10 {
        let i = r.int_n(N);
        let j = r.int_n(N);
        swapped.swap(i, j);
    }
    let mut runs16 = perm(&mut r); // a permutation cut into 16 blocks, each sorted
    for block in runs16.chunks_mut(N / 16) {
        block.sort_unstable();
    }

    let shapes: [(&str, &[i64]); 5] = [
        ("random", &random),
        ("sorted", &sorted),
        ("reversed", &reversed),
        ("sorted+10 swaps", &swapped),
        ("16 sorted runs", &runs16),
    ];

    println!("Section 1: three merge sorts, n = {}\n", commas(N));
    println!(
        "{:<16} {:<10} {:>12} {:>12} {:>7} {:>8}",
        "shape", "sort", "comparisons", "moves", "passes", "runs"
    );
    let mut all_sorted = true;
    let mut nat: HashMap<&str, (Counter, usize, usize)> = HashMap::new();
    let (mut td_random, mut bu_random, mut td_random_p) =
        (Counter::default(), Counter::default(), 0);
    for &(name, xs) in &shapes {
        let (mut a, mut b, mut c) = (xs.to_vec(), xs.to_vec(), xs.to_vec());
        let mut td = Counter::default();
        let td_p = top_down(&mut a, &mut vec![0; N], &mut td);
        let mut bu = Counter::default();
        let bu_p = bottom_up(&mut b, &mut bu);
        let mut nc = Counter::default();
        let (runs, nat_p) = natural(&mut c, &mut nc);
        all_sorted &= a.is_sorted() && b.is_sorted() && c.is_sorted();
        nat.insert(name, (nc, runs, nat_p));
        if name == "random" {
            (td_random, bu_random, td_random_p) = (td, bu, td_p);
        }
        let row = |shape: &str, sort: &str, k: Counter, p: usize, runs: &str| {
            println!(
                "{:<16} {:<10} {:>12} {:>12} {:>7} {:>8}",
                shape,
                sort,
                commas(k.cmps),
                commas(k.moves),
                p,
                runs
            );
        };
        row(name, "top-down", td, td_p, "-");
        row("", "bottom-up", bu, bu_p, "-");
        row("", "natural", nc, nat_p, &commas(runs));
    }
    let nc = |name: &str| nat[name].0.cmps;
    println!();
    println!("every output sorted: {all_sorted}");
    println!(
        "n-1 = {}; natural on sorted = {}, on reversed = {} (scan only, 0 merges)",
        commas(N - 1),
        commas(nc("sorted")),
        commas(nc("reversed"))
    );
    let (d, word) = if bu_random.cmps >= td_random.cmps {
        (bu_random.cmps - td_random.cmps, "more")
    } else {
        (td_random.cmps - bu_random.cmps, "fewer")
    };
    println!(
        "random: bottom-up makes {} {} comparisons than top-down (n is not a power of",
        commas(d),
        word
    );
    println!(
        "  two, so the pairings differ) and {:.2}x its moves -- ping-pong drops the copy back",
        bu_random.moves as f64 / td_random.moves as f64
    );
    let (_, rr, rp) = nat["random"];
    println!(
        "random: natural finds {} runs (avg length {:.2}): the scan saves {} pass(es)",
        commas(rr),
        N as f64 / rr as f64,
        td_random_p - rp
    );
    println!(
        "  and costs {} more comparisons than top-down",
        commas(nc("random") - td_random.cmps)
    );
    let (sw, swr, swp) = nat["sorted+10 swaps"];
    println!(
        "sorted+10 swaps: {} runs -> {} passes, {} comparisons = {:.1}x n: a plain merge",
        swr,
        swp,
        commas(sw.cmps),
        sw.cmps as f64 / N as f64
    );
    println!("  walks an ordered stretch one comparison per element, pass after pass");

    // Section 2: the standard library's stable sort, counted through its
    // comparator. Rust's sort_by is driftsort (core/src/slice/sort/stable/drift.rs):
    // it calls find_existing_run -- the same scan as natural() above -- and keeps
    // a run only if it is at least ~sqrt(n) long; shorter stretches go to a stable
    // quicksort, and runs are merged in powersort order.
    println!();
    println!("Section 2: sort_by (driftsort: run scan + stable quicksort), same shapes");
    println!();
    println!(
        "{:<16} {:>12} {:>12} {:>8}",
        "shape", "comparisons", "natural", "ratio"
    );
    let mut std_cmps = HashMap::new();
    for &(name, xs) in &shapes {
        let mut v = xs.to_vec();
        let mut cmps = 0;
        v.sort_by(|a, b| {
            cmps += 1;
            a.cmp(b)
        });
        std_cmps.insert(name, cmps);
        println!(
            "{:<16} {:>12} {:>12} {:>8.2}",
            name,
            commas(cmps),
            commas(nc(name)),
            cmps as f64 / nc(name) as f64
        );
    }
    println!();
    println!(
        "sorted and reversed: {} and {} comparisons -- n-1 = {}: one run found, reversed if needed",
        commas(std_cmps["sorted"]),
        commas(std_cmps["reversed"]),
        commas(N - 1)
    );
    let (s16, n16) = (std_cmps["16 sorted runs"], nc("16 sorted runs"));
    let same = if s16 == n16 {
        "the same count as"
    } else {
        "not the same as"
    };
    println!(
        "16 sorted runs: {} comparisons, {} natural() -- runs of {} are long enough to keep",
        commas(s16),
        same,
        commas(N / 16)
    );
    let (s, n10) = (std_cmps["sorted+10 swaps"], nc("sorted+10 swaps"));
    let verdict = if s < n10 { "wins" } else { "loses" };
    println!(
        "sorted+10 swaps: it {} against natural ({} vs {})",
        verdict,
        commas(s),
        commas(n10)
    );
}
```

**Rust output:**

```
Section 1: three merge sorts, n = 100,000

shape            sort        comparisons        moves  passes     runs
random           top-down      1,536,358    3,337,856      17        -
                 bottom-up     1,566,509    1,800,000      17        -
                 natural       1,610,328    1,644,944      16   41,380
sorted           top-down        815,024    3,337,856      17        -
                 bottom-up       877,968    1,800,000      17        -
                 natural          99,999            0       0        1
reversed         top-down        853,904    3,337,856      17        -
                 bottom-up       815,024    1,800,000      17        -
                 natural          99,999      100,000       0        1
sorted+10 swaps  top-down        981,341    3,337,856      17        -
                 bottom-up     1,042,676    1,800,000      17        -
                 natural         526,932      600,000       5       21
16 sorted runs   top-down      1,014,981    3,337,856      17        -
                 bottom-up     1,119,054    1,800,000      17        -
                 natural         499,956      400,000       4       16

every output sorted: true
n-1 = 99,999; natural on sorted = 99,999, on reversed = 99,999 (scan only, 0 merges)
random: bottom-up makes 30,151 more comparisons than top-down (n is not a power of
  two, so the pairings differ) and 0.54x its moves -- ping-pong drops the copy back
random: natural finds 41,380 runs (avg length 2.42): the scan saves 1 pass(es)
  and costs 73,970 more comparisons than top-down
sorted+10 swaps: 21 runs -> 5 passes, 526,932 comparisons = 5.3x n: a plain merge
  walks an ordered stretch one comparison per element, pass after pass

Section 2: sort_by (driftsort: run scan + stable quicksort), same shapes

shape             comparisons      natural    ratio
random              1,744,466    1,610,328     1.08
sorted                 99,999       99,999     1.00
reversed               99,999       99,999     1.00
sorted+10 swaps       417,453      526,932     0.79
16 sorted runs        499,956      499,956     1.00

sorted and reversed: 99,999 and 99,999 comparisons -- n-1 = 99,999: one run found, reversed if needed
16 sorted runs: 499,956 comparisons, the same count as natural() -- runs of 6,250 are long enough to keep
sorted+10 swaps: it wins against natural (417,453 vs 526,932)
```

> **Go vs Rust:** Section 1 is the Go output character for character: the same PCG permutations and the same three hand-written sorts give the same comparisons, moves and passes. Section 2 differs by design: Rust's `sort_by` is driftsort, which runs the same `find_existing_run` scan as `natural()` (read in the source, `core/src/slice/sort/shared/mod.rs`), so it makes exactly 99,999 comparisons on sorted and reversed input where Go's `SortStableFunc` makes 120,210 and 980,261. On sorted-plus-10-swaps Go's SymMerge (122,531) beats driftsort (417,453).

**Complexity:** all three Θ(n log n) worst case with Θ(n) extra space · at n = 100,000 bottom-up writes 1,800,000 elements to top-down's 3,337,856 (no copy back) · natural: exactly n − 1 = 99,999 comparisons on sorted and reversed, ⌈lg runs⌉ passes (16 runs → 4) — but 41,380 runs and 16 passes on random, and a plain merge still walks every ordered stretch, which is why `SortStableFunc`'s SymMerge beats it 122,531 to 526,932 on sorted-plus-10-swaps

---

## 8. Six pivot rules, six input shapes

`🟡 medium` · *Quicksort / pivot choice*

Quicksort's partition is fixed; what decides its complexity is the one index you hand it. This example keeps a
single Hoare quicksort and swaps only the pivot rule: first, last, middle, median-of-3, Tukey's ninther and a
seeded random index, on six input shapes. A rule goes quadratic when some shape feeds it the range minimum or
maximum again and again, so each partition peels off a constant number of elements: n(n−1)/2 comparisons and
recursion depth n − 1. Counting comparisons and depth across the 6 × 6 grid shows which rule meets its bad shape
here, and that only the random pivot stays near the expected ≈ 1.39 n lg n on every row *by construction* —
the deterministic rules that survive these six shapes survive only because no shape was built against them.

**Steps:**

1. Write one Hoare partition (pivot at `lo`, both scans stop on equal keys) and a `choose(lo, hi)` that
   implements the six rules; recurse into both sides and record the maximum depth.
2. Build six 20,000-element shapes: random, sorted, reversed, organ-pipe, sawtooth (i % 64), all-equal.
3. Print the comparison and depth tables, then each rule's worst shape and its ratio to n lg n.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"strconv"
)

// One Hoare quicksort, six ways to pick the pivot. The partition never changes;
// only the index handed to it does. Every deterministic rule has an input shape
// that feeds it the range minimum or maximum over and over, and then each
// partition peels off a constant number of elements: Θ(n²).

type rule int

const (
	first rule = iota
	last
	middle
	median3 // median of a[lo], a[mid], a[hi]
	ninther // Tukey: median of three medians of three, spread over the range
	random  // lo + IntN(len), from a seeded PCG
)

var ruleNames = []string{"first", "last", "middle", "median-of-3", "ninther", "random"}

type sorter struct {
	a        []int
	rule     rule
	rng      *rand.Rand
	cmps     int // calls to less, pivot selection included
	maxDepth int // deepest recursion level reached (the root is 1)
}

func (s *sorter) less(x, y int) bool {
	s.cmps++
	return x < y
}

// med3 returns the index of the median of a[i], a[j], a[k] in 2 or 3 comparisons.
func (s *sorter) med3(i, j, k int) int {
	if s.less(s.a[j], s.a[i]) {
		i, j = j, i // now a[i] <= a[j]
	}
	if s.less(s.a[k], s.a[j]) { // k is below the larger one
		if s.less(s.a[k], s.a[i]) {
			return i
		}
		return k
	}
	return j
}

func (s *sorter) choose(lo, hi int) int {
	mid := lo + (hi-lo)/2
	switch s.rule {
	case first:
		return lo
	case last:
		return hi
	case middle:
		return mid
	case median3:
		return s.med3(lo, mid, hi)
	case ninther:
		if hi-lo+1 < 40 { // too short to spread nine samples: plain median of 3
			return s.med3(lo, mid, hi)
		}
		d := (hi - lo + 1) / 8
		return s.med3(s.med3(lo, lo+d, lo+2*d), s.med3(mid-d, mid, mid+d), s.med3(hi-2*d, hi-d, hi))
	default:
		return lo + s.rng.IntN(hi-lo+1)
	}
}

// partition is Hoare's, with the pivot value x = a[lo]. Both scans STOP on keys
// equal to x, which is what splits a run of duplicates in the middle. It returns
// j with lo <= j < hi: a[lo..j] <= x <= a[j+1..hi]. (CLRS starts i at lo-1 and
// j at hi+1 and steps before testing; this form steps after, so no index ever
// leaves the range -- the Rust twin needs that for usize.)
func (s *sorter) partition(lo, hi int) int {
	x := s.a[lo]
	i, j := lo, hi
	for {
		for s.less(x, s.a[j]) { // skip a[j] > x
			j--
		}
		for s.less(s.a[i], x) { // skip a[i] < x
			i++
		}
		if i >= j {
			return j
		}
		s.a[i], s.a[j] = s.a[j], s.a[i]
		i++
		j--
	}
}

// sort recurses into BOTH sides, so the depth it reaches is the depth of the
// recursion tree -- the measurement. (A production sort recurses on the smaller
// side and loops on the larger, example 5.) Go's goroutine stack grows on demand,
// so depth 20,000 is fine here.
func (s *sorter) sort(lo, hi, depth int) {
	if lo >= hi {
		return
	}
	s.maxDepth = max(s.maxDepth, depth)
	p := s.choose(lo, hi)
	s.a[lo], s.a[p] = s.a[p], s.a[lo] // the pivot goes to lo for partition
	j := s.partition(lo, hi)
	s.sort(lo, j, depth+1)
	s.sort(j+1, hi, depth+1)
}

func commas(v int) string {
	s := strconv.Itoa(v)
	out := ""
	for len(s) > 3 {
		out = "," + s[len(s)-3:] + out
		s = s[:len(s)-3]
	}
	return s + out
}

func main() {
	const n = 20_000
	r := rand.New(rand.NewPCG(1, 2))

	shapeNames := []string{"random", "sorted", "reversed", "organ-pipe", "sawtooth", "all-equal"}
	shapes := make([][]int, len(shapeNames))
	for k := range shapes {
		shapes[k] = make([]int, n)
	}
	for i := range n {
		shapes[0][i] = i
		shapes[1][i] = i
		shapes[2][i] = n - 1 - i
		shapes[3][i] = min(i, n-1-i) // 0, 1, ..., n/2-1, n/2-1, ..., 0
		shapes[4][i] = i % 64
		shapes[5][i] = 0
	}
	for i := n - 1; i > 0; i-- { // Fisher-Yates over IntN
		j := r.IntN(i + 1)
		shapes[0][i], shapes[0][j] = shapes[0][j], shapes[0][i]
	}

	pivotRng := rand.New(rand.NewPCG(3, 4)) // used only by the random rule
	cmps := make([][]int, len(ruleNames))
	depth := make([][]int, len(ruleNames))
	for ru := range ruleNames {
		for sh := range shapes {
			s := sorter{a: slices.Clone(shapes[sh]), rule: rule(ru), rng: pivotRng}
			s.sort(0, n-1, 1)
			if !slices.IsSorted(s.a) {
				panic("not sorted: " + ruleNames[ru] + " on " + shapeNames[sh])
			}
			cmps[ru] = append(cmps[ru], s.cmps)
			depth[ru] = append(depth[ru], s.maxDepth)
		}
	}

	nlgn := n * math.Log2(n)
	table := func(title string, m [][]int) {
		fmt.Println(title)
		fmt.Println()
		fmt.Printf("%-12s", "pivot rule")
		for _, name := range shapeNames {
			fmt.Printf(" %12s", name)
		}
		fmt.Println()
		for ru, row := range m {
			fmt.Printf("%-12s", ruleNames[ru])
			for _, v := range row {
				fmt.Printf(" %12s", commas(v))
			}
			fmt.Println()
		}
		fmt.Println()
	}
	fmt.Printf("Hoare quicksort, n = %s, n lg n = %s, n(n-1)/2 = %s\n\n",
		commas(n), commas(int(nlgn)), commas(n*(n-1)/2))
	table("comparisons (pivot selection included):", cmps)
	table(fmt.Sprintf("maximum recursion depth (lg n = %.1f, n-1 = %s):", math.Log2(n), commas(n-1)), depth)

	// One verdict per rule, computed from the table: its worst shape, and the
	// ratio of that cell to n lg n. Above 10x is called quadratic.
	fmt.Println("worst shape per rule (comparisons / n lg n):")
	fmt.Println()
	survivors := []string{}
	for ru, row := range cmps {
		w := 0
		for sh := range row {
			if row[sh] > row[w] {
				w = sh
			}
		}
		ratio := float64(row[w]) / nlgn
		verdict := "quadratic"
		if ratio <= 10 {
			verdict = "survives all six"
			survivors = append(survivors, ruleNames[ru])
		}
		fmt.Printf("  %-12s %-11s %8.2fx   %s\n", ruleNames[ru], shapeNames[w], ratio, verdict)
	}
	fmt.Println()
	lo, hi := math.Inf(1), 0.0
	for _, v := range cmps[random] {
		lo, hi = min(lo, float64(v)/nlgn), max(hi, float64(v)/nlgn)
	}
	fmt.Printf("random pivot: %.2fx to %.2fx n lg n on all six shapes (textbook leading term: 1.39x)\n", lo, hi)
	fmt.Printf("survivors on these six shapes: %v\n", survivors)
	fixed := []string{}
	for _, name := range survivors {
		if name != "random" {
			fixed = append(fixed, name)
		}
	}
	if len(fixed) > 0 {
		fmt.Printf("%v survive only because none of these six shapes was built against them:\n", fixed)
		fmt.Println("they are still FIXED rules, and example 12 builds the input that breaks each one")
	}
}
```

**Output:**

```
Hoare quicksort, n = 20,000, n lg n = 285,754, n(n-1)/2 = 199,990,000

comparisons (pivot selection included):

pivot rule         random       sorted     reversed   organ-pipe     sawtooth    all-equal
first             435,691  200,029,998  200,039,998    1,097,575      400,497      318,430
last              458,880  200,049,997  200,039,997    1,218,977      407,625      318,430
middle            414,321      323,293      323,297   95,841,384      346,646      318,430
median-of-3       457,197      363,291      363,311      981,473      392,862      358,428
ninther           413,827      367,443      367,481      415,257      379,474      361,686
random            408,735      444,201      420,725      402,336      362,668      318,430

maximum recursion depth (lg n = 14.3, n-1 = 19,999):

pivot rule         random       sorted     reversed   organ-pipe     sawtooth    all-equal
first                  34       19,999       19,999           98           27           15
last                   37       19,999       19,999          135           28           15
middle                 34           16           16        8,188           24           15
median-of-3            32           16           16          202           24           15
ninther                25           16           16           30           20           15
random                 35           36           36           32           24           15

worst shape per rule (comparisons / n lg n):

  first        reversed      700.04x   quadratic
  last         sorted        700.08x   quadratic
  middle       organ-pipe    335.40x   quadratic
  median-of-3  organ-pipe      3.43x   survives all six
  ninther      organ-pipe      1.45x   survives all six
  random       sorted          1.55x   survives all six

random pivot: 1.11x to 1.55x n lg n on all six shapes (textbook leading term: 1.39x)
survivors on these six shapes: [median-of-3 ninther random]
[median-of-3 ninther] survive only because none of these six shapes was built against them:
they are still FIXED rules, and example 12 builds the input that breaks each one
```

**Rust:**

```rust
// One Hoare quicksort, six ways to pick the pivot. The partition never changes;
// only the index handed to it does. Every deterministic rule has an input shape
// that feeds it the range minimum or maximum over and over, and then each
// partition peels off a constant number of elements: Θ(n²).

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so the random
// shape and the random pivots are exactly the Go version's.
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
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
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

// The rule is an enum; choose() matches on it. Go uses an int with iota.
#[derive(Clone, Copy, PartialEq)]
enum Rule {
    First,
    Last,
    Middle,
    Median3, // median of a[lo], a[mid], a[hi]
    Ninther, // Tukey: median of three medians of three, spread over the range
    Random,  // lo + int_n(len), from a seeded PCG
}

impl Rule {
    const ALL: [Rule; 6] = [
        Rule::First,
        Rule::Last,
        Rule::Middle,
        Rule::Median3,
        Rule::Ninther,
        Rule::Random,
    ];

    fn name(self) -> &'static str {
        match self {
            Rule::First => "first",
            Rule::Last => "last",
            Rule::Middle => "middle",
            Rule::Median3 => "median-of-3",
            Rule::Ninther => "ninther",
            Rule::Random => "random",
        }
    }
}

struct Sorter<'a> {
    a: Vec<i64>,
    rule: Rule,
    rng: &'a mut Pcg, // borrowed: one pivot stream shared by every run, as in Go
    cmps: usize,      // calls to less, pivot selection included
    max_depth: usize, // deepest recursion level reached (the root is 1)
}

impl Sorter<'_> {
    fn less(&mut self, x: i64, y: i64) -> bool {
        self.cmps += 1;
        x < y
    }

    /// index of the median of a[i], a[j], a[k] in 2 or 3 comparisons.
    fn med3(&mut self, mut i: usize, mut j: usize, k: usize) -> usize {
        if self.less(self.a[j], self.a[i]) {
            (i, j) = (j, i); // now a[i] <= a[j]
        }
        if self.less(self.a[k], self.a[j]) {
            // k is below the larger one
            return if self.less(self.a[k], self.a[i]) {
                i
            } else {
                k
            };
        }
        j
    }

    fn choose(&mut self, lo: usize, hi: usize) -> usize {
        let mid = lo + (hi - lo) / 2;
        match self.rule {
            Rule::First => lo,
            Rule::Last => hi,
            Rule::Middle => mid,
            Rule::Median3 => self.med3(lo, mid, hi),
            Rule::Ninther if hi - lo + 1 < 40 => self.med3(lo, mid, hi), // too short for nine samples
            Rule::Ninther => {
                let d = (hi - lo + 1) / 8;
                let m1 = self.med3(lo, lo + d, lo + 2 * d);
                let m2 = self.med3(mid - d, mid, mid + d);
                let m3 = self.med3(hi - 2 * d, hi - d, hi);
                self.med3(m1, m2, m3)
            }
            Rule::Random => lo + self.rng.int_n(hi - lo + 1),
        }
    }

    // Hoare's partition with the pivot value x = a[lo]. Both scans STOP on keys
    // equal to x, which splits a run of duplicates in the middle. Returns j with
    // lo <= j < hi: a[lo..=j] <= x <= a[j+1..=hi]. CLRS starts i at lo-1, which
    // is an underflow panic for usize when lo = 0; this form steps AFTER the swap
    // instead, so no index leaves the range (the Go version uses the same form).
    fn partition(&mut self, lo: usize, hi: usize) -> usize {
        let x = self.a[lo];
        let (mut i, mut j) = (lo, hi);
        loop {
            while self.less(x, self.a[j]) {
                j -= 1; // skip a[j] > x
            }
            while self.less(self.a[i], x) {
                i += 1; // skip a[i] < x
            }
            if i >= j {
                return j;
            }
            self.a.swap(i, j);
            i += 1;
            j -= 1;
        }
    }

    // Recurse into BOTH sides: the depth reached is the measurement. Depth can
    // reach n-1 = 19,999 frames, so main() runs this on a thread with a 64 MiB
    // stack -- Go's goroutine stack grows by itself, a Rust thread's does not.
    fn sort(&mut self, lo: usize, hi: usize, depth: usize) {
        if lo >= hi {
            return;
        }
        self.max_depth = self.max_depth.max(depth);
        let p = self.choose(lo, hi);
        self.a.swap(lo, p); // the pivot goes to lo for partition
        let j = self.partition(lo, hi);
        self.sort(lo, j, depth + 1);
        self.sort(j + 1, hi, depth + 1);
    }
}

fn commas(v: usize) -> String {
    let s = v.to_string();
    let mut out = String::new();
    for (k, ch) in s.chars().enumerate() {
        if k > 0 && (s.len() - k).is_multiple_of(3) {
            out.push(',');
        }
        out.push(ch);
    }
    out
}

fn run() {
    const N: usize = 20_000;
    let mut r = Pcg::new(1, 2);

    let shape_names = [
        "random",
        "sorted",
        "reversed",
        "organ-pipe",
        "sawtooth",
        "all-equal",
    ];
    let n = N as i64;
    let mut random: Vec<i64> = (0..n).collect();
    for i in (1..N).rev() {
        let j = r.int_n(i + 1); // Fisher-Yates over int_n
        random.swap(i, j);
    }
    let shapes: [Vec<i64>; 6] = [
        random,
        (0..n).collect(),
        (0..n).rev().collect(),
        (0..n).map(|i| i.min(n - 1 - i)).collect(), // 0, 1, ..., n/2-1, n/2-1, ..., 0
        (0..n).map(|i| i % 64).collect(),
        vec![0; N],
    ];

    let mut pivot_rng = Pcg::new(3, 4); // used only by the random rule
    let mut cmps = vec![vec![]; Rule::ALL.len()];
    let mut depth = vec![vec![]; Rule::ALL.len()];
    for (ru, &rule) in Rule::ALL.iter().enumerate() {
        for (sh, shape) in shapes.iter().enumerate() {
            let mut s = Sorter {
                a: shape.clone(),
                rule,
                rng: &mut pivot_rng,
                cmps: 0,
                max_depth: 0,
            };
            s.sort(0, N - 1, 1);
            assert!(
                s.a.is_sorted(),
                "not sorted: {} on {}",
                rule.name(),
                shape_names[sh]
            );
            cmps[ru].push(s.cmps);
            depth[ru].push(s.max_depth);
        }
    }

    let nlgn = N as f64 * (N as f64).log2();
    let table = |title: &str, m: &[Vec<usize>]| {
        println!("{title}\n");
        print!("{:<12}", "pivot rule");
        for name in shape_names {
            print!(" {name:>12}");
        }
        println!();
        for (rule, row) in Rule::ALL.iter().zip(m) {
            print!("{:<12}", rule.name());
            for &v in row {
                print!(" {:>12}", commas(v));
            }
            println!();
        }
        println!();
    };
    println!(
        "Hoare quicksort, n = {}, n lg n = {}, n(n-1)/2 = {}\n",
        commas(N),
        commas(nlgn as usize),
        commas(N * (N - 1) / 2)
    );
    table("comparisons (pivot selection included):", &cmps);
    table(
        &format!(
            "maximum recursion depth (lg n = {:.1}, n-1 = {}):",
            (N as f64).log2(),
            commas(N - 1)
        ),
        &depth,
    );

    // One verdict per rule, computed from the table: its worst shape, and the
    // ratio of that cell to n lg n. Above 10x is called quadratic.
    println!("worst shape per rule (comparisons / n lg n):\n");
    let mut survivors = vec![];
    for (rule, row) in Rule::ALL.iter().zip(&cmps) {
        // first maximum, like Go's strict `>` scan (max_by_key would keep the last)
        let w = (0..row.len()).fold(0, |w, sh| if row[sh] > row[w] { sh } else { w });
        let ratio = row[w] as f64 / nlgn;
        let verdict = if ratio <= 10.0 {
            survivors.push(*rule);
            "survives all six"
        } else {
            "quadratic"
        };
        println!(
            "  {:<12} {:<11} {:>8.2}x   {}",
            rule.name(),
            shape_names[w],
            ratio,
            verdict
        );
    }
    println!();
    let random_row = Rule::ALL.iter().position(|&r| r == Rule::Random).unwrap();
    let ratios = cmps[random_row].iter().map(|&v| v as f64 / nlgn);
    let lo = ratios.clone().fold(f64::INFINITY, f64::min);
    let hi = ratios.fold(0.0, f64::max);
    println!(
        "random pivot: {lo:.2}x to {hi:.2}x n lg n on all six shapes (textbook leading term: 1.39x)"
    );
    let names = |rs: &[Rule]| rs.iter().map(|r| r.name()).collect::<Vec<_>>().join(" ");
    println!("survivors on these six shapes: [{}]", names(&survivors));
    let fixed: Vec<Rule> = survivors
        .into_iter()
        .filter(|&r| r != Rule::Random)
        .collect();
    if !fixed.is_empty() {
        println!(
            "[{}] survive only because none of these six shapes was built against them:",
            names(&fixed)
        );
        println!(
            "they are still FIXED rules, and example 12 builds the input that breaks each one"
        );
    }
}

fn main() {
    // the deep recursion (19,999 frames) needs more than the main thread's
    // fixed stack may give; a spawned thread's size is ours to choose
    std::thread::Builder::new()
        .stack_size(64 << 20)
        .spawn(run)
        .expect("spawn")
        .join()
        .expect("sorting thread panicked");
}
```

**Rust output:**

```
Hoare quicksort, n = 20,000, n lg n = 285,754, n(n-1)/2 = 199,990,000

comparisons (pivot selection included):

pivot rule         random       sorted     reversed   organ-pipe     sawtooth    all-equal
first             435,691  200,029,998  200,039,998    1,097,575      400,497      318,430
last              458,880  200,049,997  200,039,997    1,218,977      407,625      318,430
middle            414,321      323,293      323,297   95,841,384      346,646      318,430
median-of-3       457,197      363,291      363,311      981,473      392,862      358,428
ninther           413,827      367,443      367,481      415,257      379,474      361,686
random            408,735      444,201      420,725      402,336      362,668      318,430

maximum recursion depth (lg n = 14.3, n-1 = 19,999):

pivot rule         random       sorted     reversed   organ-pipe     sawtooth    all-equal
first                  34       19,999       19,999           98           27           15
last                   37       19,999       19,999          135           28           15
middle                 34           16           16        8,188           24           15
median-of-3            32           16           16          202           24           15
ninther                25           16           16           30           20           15
random                 35           36           36           32           24           15

worst shape per rule (comparisons / n lg n):

  first        reversed      700.04x   quadratic
  last         sorted        700.08x   quadratic
  middle       organ-pipe    335.40x   quadratic
  median-of-3  organ-pipe      3.43x   survives all six
  ninther      organ-pipe      1.45x   survives all six
  random       sorted          1.55x   survives all six

random pivot: 1.11x to 1.55x n lg n on all six shapes (textbook leading term: 1.39x)
survivors on these six shapes: [median-of-3 ninther random]
[median-of-3 ninther] survive only because none of these six shapes was built against them:
they are still FIXED rules, and example 12 builds the input that breaks each one
```

> **Go vs Rust:** The output is the Go output character for character: same PCG shape and pivot stream, same partition, same counts and depths. The structural differences are in the code: the rule is an `enum` matched in `choose` (a guard arm picks plain median-of-3 for short ranges), CLRS's `i = lo - 1` would underflow a `usize` so both versions step after the swap instead, and the sort runs on a thread with a 64 MiB stack because the 19,999-deep recursion of the first/last rules relies on a stack Go grows by itself and Rust does not.

**Complexity:** random pivot ≈ 1.39 n lg n expected on every input (measured 1.11–1.55× n lg n here) · first and last: ≈ 200 million comparisons and depth 19,999 on sorted/reversed (700× n lg n) · middle: 95.8 million and depth 8,188 on organ-pipe · median-of-3 (3.43× on organ-pipe) and the ninther (1.45×) survive these six shapes, but each deterministic rule is Θ(n²) on some input — example 12 constructs it

---

## 9. Three-way partition for duplicates

`🟡 medium` · *Quicksort / Θ(n log k)*

Real data is full of repeated keys, and a two-way partition handles them badly. Lomuto sends every key equal to
the pivot to one side, so a block of m equal keys costs m(m−1)/2 comparisons: quadratic on all-equal input.
Hoare splits equal keys down the middle and keeps Θ(n log n), but it compares those equal keys again at every
level below. Dijkstra's **3-way partition** (`lt`, `i`, `gt`) splits the range into `< p`, `== p` and `> p`,
and the `== p` block is final: it is never touched again. With k distinct values that is about Θ(n log k)
comparisons, and a single Θ(n) pass when all keys are equal. Both standard libraries contain a version of the
same idea (Go's `partitionEqual`, Rust's ancestor-pivot check), and the counts below put them side by side.

**Steps:**

1. Write Lomuto, Hoare and 3-way quicksorts that share one pivot rule (a random index from a fresh seeded PCG)
   and recurse on the smaller side, so only the partition differs.
2. Sort 20,000 values drawn from `IntN(k)` for k = 1, 2, 10, 100, 1,000 and 20,000, counting comparator calls
   (a three-way compare counts as one).
3. Compare the three columns with n·lg k, then count the standard library's unstable sort on the same inputs.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"strconv"
)

// Three quicksorts that differ ONLY in the partition, run on input with k
// distinct values. All three use the same pivot rule -- a random index from a
// fresh PCG(3, 4) per sort -- and all three recurse on the smaller side and loop
// on the larger, so the stack stays O(log n) even when the time does not.
// Ranges are half-open, a[lo:hi].
type sorter struct {
	a    []int
	rng  *rand.Rand
	cmps int // comparator calls; a three-way compare counts as ONE
}

func (s *sorter) less(x, y int) bool   { s.cmps++; return x < y }
func (s *sorter) compare(x, y int) int { s.cmps++; return cmp.Compare(x, y) }

func (s *sorter) pivotTo(lo, hi, dst int) { // move a random element of a[lo:hi] to dst
	p := lo + s.rng.IntN(hi-lo)
	s.a[p], s.a[dst] = s.a[dst], s.a[p]
}

// Lomuto: elements < x go left. An element EQUAL to x stays right, so on a block
// of equal keys nothing moves and the pivot lands at lo: a split of 0 | m-1.
func (s *sorter) lomuto(lo, hi int) {
	for hi-lo > 1 {
		s.pivotTo(lo, hi, hi-1)
		x, i := s.a[hi-1], lo
		for j := lo; j < hi-1; j++ {
			if s.less(s.a[j], x) {
				s.a[i], s.a[j] = s.a[j], s.a[i]
				i++
			}
		}
		s.a[i], s.a[hi-1] = s.a[hi-1], s.a[i] // pivot to its final index i
		if i-lo < hi-i-1 {
			s.lomuto(lo, i)
			lo = i + 1
		} else {
			s.lomuto(i+1, hi)
			hi = i
		}
	}
}

// Hoare: both scans STOP on keys equal to x and swap them, so a block of equal
// keys splits in the middle -- Θ(n log n) comparisons, but the equal keys are
// compared again at every level below.
func (s *sorter) hoare(lo, hi int) {
	for hi-lo > 1 {
		s.pivotTo(lo, hi, lo)
		x, i, j := s.a[lo], lo, hi-1
		for {
			for s.less(x, s.a[j]) {
				j--
			}
			for s.less(s.a[i], x) {
				i++
			}
			if i >= j {
				break
			}
			s.a[i], s.a[j] = s.a[j], s.a[i]
			i++
			j--
		}
		// a[lo:j+1] <= x <= a[j+1:hi]
		if j+1-lo < hi-j-1 {
			s.hoare(lo, j+1)
			lo = j + 1
		} else {
			s.hoare(j+1, hi)
			hi = j + 1
		}
	}
}

// threeWay is Dijkstra's Dutch national flag:
//
//	a[lo:lt] < x    a[lt:i] == x    a[i:gt] unseen    a[gt:hi] > x
//
// One compare per element per pass. The == block is final and never touched
// again; on all-equal input the first pass ends with lt = lo, gt = hi: done.
func (s *sorter) threeWay(lo, hi int) {
	for hi-lo > 1 {
		s.pivotTo(lo, hi, lo)
		x, lt, i, gt := s.a[lo], lo, lo+1, hi
		for i < gt {
			switch c := s.compare(s.a[i], x); {
			case c < 0:
				s.a[lt], s.a[i] = s.a[i], s.a[lt]
				lt++
				i++
			case c > 0:
				gt--
				s.a[i], s.a[gt] = s.a[gt], s.a[i]
			default:
				i++
			}
		}
		if lt-lo < hi-gt {
			s.threeWay(lo, lt)
			lo = gt
		} else {
			s.threeWay(gt, hi)
			hi = lt
		}
	}
}

func commas(v int) string {
	s := strconv.Itoa(v)
	out := ""
	for len(s) > 3 {
		out = "," + s[len(s)-3:] + out
		s = s[:len(s)-3]
	}
	return s + out
}

func main() {
	const n = 20_000
	ks := []int{1, 2, 10, 100, 1_000, 20_000}
	r := rand.New(rand.NewPCG(1, 2))

	inputs := make([][]int, len(ks))
	for t, k := range ks {
		inputs[t] = make([]int, n)
		for i := range inputs[t] {
			inputs[t][i] = r.IntN(k)
		}
	}

	run := func(xs []int, sort func(s *sorter)) int {
		s := &sorter{a: slices.Clone(xs), rng: rand.New(rand.NewPCG(3, 4))}
		sort(s)
		if !slices.IsSorted(s.a) {
			panic("not sorted")
		}
		return s.cmps
	}

	fmt.Printf("n = %s, values IntN(k), random pivot (PCG) for all three partitions\n\n", commas(n))
	fmt.Printf("%7s %9s %13s %11s %11s %11s %7s\n",
		"k", "distinct", "Lomuto", "Hoare", "3-way", "n lg k", "3w/nlgk")
	three := make([]int, len(ks))
	var lom1, hoare1 int
	for t, k := range ks {
		distinct := len(slices.Compact(slices.Sorted(slices.Values(inputs[t]))))
		lo := run(inputs[t], func(s *sorter) { s.lomuto(0, n) })
		ho := run(inputs[t], func(s *sorter) { s.hoare(0, n) })
		three[t] = run(inputs[t], func(s *sorter) { s.threeWay(0, n) })
		if k == 1 {
			lom1, hoare1 = lo, ho
		}
		nlgk := n * math.Log2(float64(k))
		ratio := "-"
		if k > 1 {
			ratio = fmt.Sprintf("%.2f", float64(three[t])/nlgk)
		}
		fmt.Printf("%7s %9s %13s %11s %11s %11s %7s\n", commas(k), commas(distinct),
			commas(lo), commas(ho), commas(three[t]), commas(int(nlgk)), ratio)
	}
	fmt.Println()
	fmt.Printf("k = 1: Lomuto %s = n(n-1)/2 = %s: %v -- every partition splits 0 | m-1\n",
		commas(lom1), commas(n*(n-1)/2), lom1 == n*(n-1)/2)
	fmt.Printf("k = 1: 3-way %s = n-1: %v -- one pass, then nothing is left to sort\n",
		commas(three[0]), three[0] == n-1)
	fmt.Printf("k = 1: Hoare %s = %.2f n lg n -- correct complexity, but it re-sorts equal keys\n",
		commas(hoare1), float64(hoare1)/(n*math.Log2(n)))
	fmt.Printf("k = 20,000 (mostly distinct): 3-way %s vs k = 10: %s -- %.1fx, for lg k %.1fx larger\n",
		commas(three[5]), commas(three[2]), float64(three[5])/float64(three[2]),
		math.Log2(20_000)/math.Log2(10))

	// Section 2: the standard library on the same inputs. pdqsort has the 3-way
	// idea built in: when the element just before a range is not less than the
	// pivot, the pivot is that range's minimum, so it partitions into == and >
	// and drops the == block (partitionEqualCmpFunc, src/slices/zsortanyfunc.go).
	// All-equal input is also SORTED input, which pdqsort detects separately, so
	// the k = 1 row does not show which of the two shortcuts did the work.
	fmt.Println()
	fmt.Println("Section 2: slices.SortFunc (pdqsort) on the same inputs")
	fmt.Println()
	fmt.Printf("%7s %13s %11s %8s\n", "k", "SortFunc", "3-way", "ratio")
	for t, k := range ks {
		xs := slices.Clone(inputs[t])
		c := 0
		slices.SortFunc(xs, func(a, b int) int { c++; return cmp.Compare(a, b) })
		fmt.Printf("%7s %13s %11s %8.2f\n", commas(k), commas(c), commas(three[t]),
			float64(c)/float64(three[t]))
	}
}
```

**Output:**

```
n = 20,000, values IntN(k), random pivot (PCG) for all three partitions

      k  distinct        Lomuto       Hoare       3-way      n lg k 3w/nlgk
      1         1   199,990,000     318,430      19,999           0       -
      2         2   100,011,848     322,043      29,955      20,000    1.50
     10        10    20,068,763     340,401      62,249      66,438    0.94
    100       100     2,180,967     352,757     136,468     132,877    1.03
  1,000     1,000       441,264     393,651     248,721     199,315    1.25
 20,000    12,581       329,863     409,887     318,519     285,754    1.11

k = 1: Lomuto 199,990,000 = n(n-1)/2 = 199,990,000: true -- every partition splits 0 | m-1
k = 1: 3-way 19,999 = n-1: true -- one pass, then nothing is left to sort
k = 1: Hoare 318,430 = 1.11 n lg n -- correct complexity, but it re-sorts equal keys
k = 20,000 (mostly distinct): 3-way 318,519 vs k = 10: 62,249 -- 5.1x, for lg k 4.3x larger

Section 2: slices.SortFunc (pdqsort) on the same inputs

      k      SortFunc       3-way    ratio
      1        20,011      19,999     1.00
      2        50,088      29,955     1.67
     10        90,305      62,249     1.45
    100       162,286     136,468     1.19
  1,000       244,430     248,721     0.98
 20,000       295,366     318,519     0.93
```

**Rust:**

```rust
use std::cmp::Ordering;

// Three quicksorts that differ ONLY in the partition, run on input with k
// distinct values. All three use the same pivot rule -- a random index from a
// fresh Pcg(3, 4) per sort -- and all three recurse on the smaller side and loop
// on the larger, so the stack stays O(log n) even when the time does not.
// Ranges are half-open, a[lo..hi], which also keeps every usize index >= 0.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so the inputs
// and the pivots are exactly the Go version's.
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
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
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

struct Sorter {
    a: Vec<i64>,
    rng: Pcg,
    cmps: usize, // comparator calls; a three-way compare counts as ONE
}

impl Sorter {
    fn less(&mut self, x: i64, y: i64) -> bool {
        self.cmps += 1;
        x < y
    }

    fn compare(&mut self, x: i64, y: i64) -> Ordering {
        self.cmps += 1;
        x.cmp(&y)
    }

    fn pivot_to(&mut self, lo: usize, hi: usize, dst: usize) {
        let p = lo + self.rng.int_n(hi - lo); // a random element of a[lo..hi] to dst
        self.a.swap(p, dst);
    }

    // Lomuto: elements < x go left. An element EQUAL to x stays right, so on a
    // block of equal keys nothing moves and the pivot lands at lo: 0 | m-1.
    fn lomuto(&mut self, mut lo: usize, mut hi: usize) {
        while hi - lo > 1 {
            self.pivot_to(lo, hi, hi - 1);
            let (x, mut i) = (self.a[hi - 1], lo);
            for j in lo..hi - 1 {
                if self.less(self.a[j], x) {
                    self.a.swap(i, j);
                    i += 1;
                }
            }
            self.a.swap(i, hi - 1); // pivot to its final index i
            if i - lo < hi - i - 1 {
                self.lomuto(lo, i);
                lo = i + 1;
            } else {
                self.lomuto(i + 1, hi);
                hi = i;
            }
        }
    }

    // Hoare: both scans STOP on keys equal to x and swap them, so a block of
    // equal keys splits in the middle -- Θ(n log n), but the equal keys are
    // compared again at every level below.
    fn hoare(&mut self, mut lo: usize, mut hi: usize) {
        while hi - lo > 1 {
            self.pivot_to(lo, hi, lo);
            let (x, mut i, mut j) = (self.a[lo], lo, hi - 1);
            loop {
                while self.less(x, self.a[j]) {
                    j -= 1;
                }
                while self.less(self.a[i], x) {
                    i += 1;
                }
                if i >= j {
                    break;
                }
                self.a.swap(i, j);
                i += 1;
                j -= 1;
            }
            // a[lo..=j] <= x <= a[j+1..hi]
            if j + 1 - lo < hi - j - 1 {
                self.hoare(lo, j + 1);
                lo = j + 1;
            } else {
                self.hoare(j + 1, hi);
                hi = j + 1;
            }
        }
    }

    // three_way is Dijkstra's Dutch national flag:
    //   a[lo..lt] < x    a[lt..i] == x    a[i..gt] unseen    a[gt..hi] > x
    // One compare per element per pass, and Ordering makes the three cases a
    // match the compiler checks for exhaustiveness (Go switches on an int's sign).
    // The == block is final; on all-equal input the first pass ends lt = lo,
    // gt = hi: done.
    fn three_way(&mut self, mut lo: usize, mut hi: usize) {
        while hi - lo > 1 {
            self.pivot_to(lo, hi, lo);
            let (x, mut lt, mut i, mut gt) = (self.a[lo], lo, lo + 1, hi);
            while i < gt {
                match self.compare(self.a[i], x) {
                    Ordering::Less => {
                        self.a.swap(lt, i);
                        lt += 1;
                        i += 1;
                    }
                    Ordering::Greater => {
                        gt -= 1;
                        self.a.swap(i, gt);
                    }
                    Ordering::Equal => i += 1,
                }
            }
            if lt - lo < hi - gt {
                self.three_way(lo, lt);
                lo = gt;
            } else {
                self.three_way(gt, hi);
                hi = lt;
            }
        }
    }
}

fn commas(v: usize) -> String {
    let s = v.to_string();
    let mut out = String::new();
    for (k, ch) in s.chars().enumerate() {
        if k > 0 && (s.len() - k).is_multiple_of(3) {
            out.push(',');
        }
        out.push(ch);
    }
    out
}

fn run(xs: &[i64], sort: fn(&mut Sorter, usize, usize)) -> usize {
    let mut s = Sorter {
        a: xs.to_vec(),
        rng: Pcg::new(3, 4),
        cmps: 0,
    };
    sort(&mut s, 0, xs.len());
    assert!(s.a.is_sorted(), "not sorted");
    s.cmps
}

fn main() {
    const N: usize = 20_000;
    let ks = [1, 2, 10, 100, 1_000, 20_000];
    let mut r = Pcg::new(1, 2);
    let inputs: Vec<Vec<i64>> = ks
        .iter()
        .map(|&k| (0..N).map(|_| r.int_n(k) as i64).collect())
        .collect();

    println!(
        "n = {}, values IntN(k), random pivot (PCG) for all three partitions\n",
        commas(N)
    );
    println!(
        "{:>7} {:>9} {:>13} {:>11} {:>11} {:>11} {:>7}",
        "k", "distinct", "Lomuto", "Hoare", "3-way", "n lg k", "3w/nlgk"
    );
    let mut three = vec![];
    let (mut lom1, mut hoare1) = (0, 0);
    for (xs, &k) in inputs.iter().zip(&ks) {
        let mut d = xs.clone();
        d.sort_unstable();
        d.dedup();
        let lo = run(xs, Sorter::lomuto);
        let ho = run(xs, Sorter::hoare);
        let tw = run(xs, Sorter::three_way);
        three.push(tw);
        if k == 1 {
            (lom1, hoare1) = (lo, ho);
        }
        let nlgk = N as f64 * (k as f64).log2();
        let ratio = if k > 1 {
            format!("{:.2}", tw as f64 / nlgk)
        } else {
            "-".to_string()
        };
        println!(
            "{:>7} {:>9} {:>13} {:>11} {:>11} {:>11} {:>7}",
            commas(k),
            commas(d.len()),
            commas(lo),
            commas(ho),
            commas(tw),
            commas(nlgk as usize),
            ratio
        );
    }
    println!();
    println!(
        "k = 1: Lomuto {} = n(n-1)/2 = {}: {} -- every partition splits 0 | m-1",
        commas(lom1),
        commas(N * (N - 1) / 2),
        lom1 == N * (N - 1) / 2
    );
    println!(
        "k = 1: 3-way {} = n-1: {} -- one pass, then nothing is left to sort",
        commas(three[0]),
        three[0] == N - 1
    );
    println!(
        "k = 1: Hoare {} = {:.2} n lg n -- correct complexity, but it re-sorts equal keys",
        commas(hoare1),
        hoare1 as f64 / (N as f64 * (N as f64).log2())
    );
    println!(
        "k = 20,000 (mostly distinct): 3-way {} vs k = 10: {} -- {:.1}x, for lg k {:.1}x larger",
        commas(three[5]),
        commas(three[2]),
        three[5] as f64 / three[2] as f64,
        20_000f64.log2() / 10f64.log2()
    );

    // Section 2: the standard library on the same inputs. ipnsort has the 3-way
    // idea built in: quicksort() carries the pivot just left of the range (`ancestor_pivot`,
    // core/src/slice/sort/unstable/quicksort.rs); if it is not less than the new
    // pivot, the new pivot is the range minimum, so it partitions off the
    // elements equal to it and never recurses into them. All-equal input is also
    // one ascending run, which ipnsort detects first (n-1 comparisons).
    println!();
    println!("Section 2: sort_unstable_by (ipnsort) on the same inputs");
    println!();
    println!(
        "{:>7} {:>13} {:>11} {:>8}",
        "k", "sort_unstable", "3-way", "ratio"
    );
    for ((xs, &k), &tw) in inputs.iter().zip(&ks).zip(&three) {
        let mut v = xs.clone();
        let mut c = 0;
        v.sort_unstable_by(|a, b| {
            c += 1;
            a.cmp(b)
        });
        println!(
            "{:>7} {:>13} {:>11} {:>8.2}",
            commas(k),
            commas(c),
            commas(tw),
            c as f64 / tw as f64
        );
    }
}
```

**Rust output:**

```
n = 20,000, values IntN(k), random pivot (PCG) for all three partitions

      k  distinct        Lomuto       Hoare       3-way      n lg k 3w/nlgk
      1         1   199,990,000     318,430      19,999           0       -
      2         2   100,011,848     322,043      29,955      20,000    1.50
     10        10    20,068,763     340,401      62,249      66,438    0.94
    100       100     2,180,967     352,757     136,468     132,877    1.03
  1,000     1,000       441,264     393,651     248,721     199,315    1.25
 20,000    12,581       329,863     409,887     318,519     285,754    1.11

k = 1: Lomuto 199,990,000 = n(n-1)/2 = 199,990,000: true -- every partition splits 0 | m-1
k = 1: 3-way 19,999 = n-1: true -- one pass, then nothing is left to sort
k = 1: Hoare 318,430 = 1.11 n lg n -- correct complexity, but it re-sorts equal keys
k = 20,000 (mostly distinct): 3-way 318,519 vs k = 10: 62,249 -- 5.1x, for lg k 4.3x larger

Section 2: sort_unstable_by (ipnsort) on the same inputs

      k sort_unstable       3-way    ratio
      1        19,999      19,999     1.00
      2        50,514      29,955     1.69
     10        93,072      62,249     1.50
    100       162,176     136,468     1.19
  1,000       271,648     248,721     1.09
 20,000       296,556     318,519     0.93
```

> **Go vs Rust:** Section 1 is the Go output character for character: same `IntN(k)` inputs, same pivot stream, same counts. Section 2 differs by design: `sort_unstable_by` is ipnsort, which first checks whether the whole slice is one run, so all-equal input costs exactly 19,999 = n − 1 (Go's pdqsort: 20,011); on the other rows the two stay within 12% of each other (the largest gap is k = 1,000: 271,648 vs Go's 244,430), each with its own form of the equal-pivot shortcut (`ancestor_pivot` in Rust, `partitionEqual` in Go — read in the source, not isolated by this program).

**Complexity:** 3-way: Θ(n) on all-equal input (exactly n − 1 = 19,999 comparisons) and ≈ Θ(n log k) for k distinct values (0.94–1.50 × n lg k measured) · Lomuto: Θ(n²) at k = 1 (exactly n(n−1)/2 = 199,990,000) and 100 million at k = 2 · Hoare: Θ(n log n) regardless of k (318,430 at k = 1, 409,887 at k = 20,000) · all three O(log n) stack, smaller side first

---

## 10. Quickselect: the k-th smallest without sorting

`🟡 medium` · *Selection / expected Θ(n)*

To find the k-th smallest you do not need the other n − 1 elements in order, only on the correct side of it.
Quickselect partitions once, keeps the side that holds index k and drops the other for good, so a random
pivot costs n + n/2 + n/4 + … on average: **expected Θ(n)**, about 2n comparisons for the minimum and
2(1 + ln 2)n ≈ 3.39n for the median, against the ~n lg n of sorting first. It is still quicksort's partition,
so it inherits quicksort's worst case: a fixed pivot on sorted input makes it Θ(n²). Go has no selection
function in its standard library, so this is the code you would write; Rust ships it as `select_nth_unstable`.

**Steps:**

1. Write an iterative quickselect over a Lomuto partition (exactly len − 1 comparisons per round) with a
   random pivot from the seeded PCG, and average its comparisons over 200 random permutations of 100,000.
2. Compare it on one permutation with sort-then-index, use it for top-10 (select k = 9, sort the 10 on the
   left), and run it once with a first-element pivot on sorted input.
3. Read the output: the averages next to Knuth's formula (with their spread), the ≈ 6× gap to sorting, and
   the exact n²-shaped count of the fixed pivot.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
)

// Quickselect finds the k-th smallest element (0-based) without sorting:
// partition once, then continue into ONLY the side that holds index k. The
// other side is never looked at again, so the work is n + n/2 + n/4 + ... on
// average -- linear -- instead of quicksort's n per level for lg n levels.

// partition is Lomuto's scheme on xs[lo:hi+1] with the pivot already at hi.
// It makes exactly (hi - lo) comparisons -- one per non-pivot element -- and
// returns the pivot's final index: everything left of it is smaller.
func partition(xs []int, lo, hi int, cmps *int) int {
	p := xs[hi]
	i := lo
	for j := lo; j < hi; j++ {
		*cmps++
		if xs[j] < p {
			xs[i], xs[j] = xs[j], xs[i]
			i++
		}
	}
	xs[i], xs[hi] = xs[hi], xs[i]
	return i
}

// quickselect is iterative (a loop, no recursion: O(1) extra space). With
// randomPivot it swaps a random element of the range to the end first; without
// it, it uses the FIRST element -- the textbook rule that sorted input defeats.
func quickselect(xs []int, k int, r *rand.Rand, randomPivot bool) (int, int) {
	cmps := 0
	lo, hi := 0, len(xs)-1
	for lo < hi {
		pv := lo
		if randomPivot {
			pv = lo + r.IntN(hi-lo+1)
		}
		xs[pv], xs[hi] = xs[hi], xs[pv]
		m := partition(xs, lo, hi, &cmps)
		switch {
		case k < m:
			hi = m - 1 // k is in the left part; drop the right one for good
		case k > m:
			lo = m + 1 // k is in the right part
		default:
			return xs[k], cmps // the pivot landed exactly on k
		}
	}
	return xs[k], cmps
}

// mergeSort (top-down, one scratch buffer) is the "sort, then index" baseline.
func mergeSort(xs, buf []int, cmps *int) {
	if len(xs) < 2 {
		return
	}
	mid := len(xs) / 2
	mergeSort(xs[:mid], buf[:mid], cmps)
	mergeSort(xs[mid:], buf[mid:], cmps)
	i, j, o := 0, mid, 0
	for i < mid && j < len(xs) {
		*cmps++
		if xs[j] < xs[i] {
			buf[o] = xs[j]
			j++
		} else {
			buf[o] = xs[i]
			i++
		}
		o++
	}
	o += copy(buf[o:], xs[i:mid])
	copy(buf[o:], xs[j:])
	copy(xs, buf[:len(xs)])
}

// shuffled returns a Fisher-Yates permutation of 0..n-1 driven by IntN only,
// so the Rust twin (with its PCG port) builds exactly the same arrays.
func shuffled(n int, r *rand.Rand) []int {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i
	}
	for i := n - 1; i > 0; i-- {
		j := r.IntN(i + 1)
		xs[i], xs[j] = xs[j], xs[i]
	}
	return xs
}

// theory is Knuth's expected comparisons per element for random-pivot
// quickselect, to leading order: 2 + 2a ln(1/a) + 2(1-a) ln(1/(1-a)), a = k/n.
func theory(k, n int) float64 {
	a := float64(k) / float64(n)
	t := 2.0
	if a > 0 {
		t += 2 * a * math.Log(1/a)
	}
	if a < 1 {
		t += 2 * (1 - a) * math.Log(1/(1-a))
	}
	return t
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	const n, trials = 100_000, 200
	ks := []int{0, n / 10, n / 4, n / 2}

	fmt.Printf("1) random-pivot quickselect, n = %d, average of %d permutations\n", n, trials)
	sums := make([]float64, len(ks)) // sum of comparisons/n
	sqs := make([]float64, len(ks))  // sum of squares, for the spread
	work := make([]int, n)
	for t := 0; t < trials; t++ {
		xs := shuffled(n, r)
		for i, k := range ks {
			copy(work, xs)
			v, c := quickselect(work, k, r, true)
			if v != k { // a permutation of 0..n-1: the k-th smallest IS k
				panic("wrong answer")
			}
			x := float64(c) / n
			sums[i] += x
			sqs[i] += x * x
		}
	}
	// One run varies a lot (its standard deviation is about n comparisons), so
	// print the mean with +-2 standard errors and check the theory against that.
	fmt.Println("        k   comparisons/n   +-2 s.e.   theory   theory inside?")
	for i, k := range ks {
		mean := sums[i] / trials
		se := math.Sqrt((sqs[i]/trials - mean*mean) / trials)
		th := theory(k, n)
		fmt.Printf("  %7d   %13.3f   %8.3f   %6.3f   %v\n", k, mean, 2*se, th, math.Abs(mean-th) <= 2*se)
	}

	fmt.Println("\n2) the median of one permutation: sort-and-index vs quickselect")
	xs := shuffled(n, r)
	sorted := slices.Clone(xs)
	ms := 0
	mergeSort(sorted, make([]int, n), &ms)
	work = slices.Clone(xs)
	med, qs := quickselect(work, n/2, r, true)
	fmt.Printf("  merge sort, then index : %9d comparisons -> %d\n", ms, sorted[n/2])
	fmt.Printf("  quickselect            : %9d comparisons -> %d\n", qs, med)
	fmt.Printf("  quickselect does %.1fx fewer comparisons\n", float64(ms)/float64(qs))

	fmt.Println("\n3) top-10: select k = 9, then sort only the 10 on the left")
	work = slices.Clone(xs)
	_, tc := quickselect(work, 9, r, true)
	top := work[:10] // left of k: all <= xs[9], in no particular order
	ins := 0         // insertion sort on just these 10
	for i := 1; i < len(top); i++ {
		for j := i; j > 0; j-- {
			ins++
			if top[j] >= top[j-1] {
				break
			}
			top[j], top[j-1] = top[j-1], top[j]
		}
	}
	fmt.Printf("  %v  select %d + sort %d = %d comparisons\n", top, tc, ins, tc+ins)
	fmt.Printf("  equal to the full sort's first 10: %v (full sort: %d comparisons)\n",
		slices.Equal(top, sorted[:10]), ms)

	fmt.Println("\n4) first-element pivot, already-sorted input, median")
	const m = 10_000
	asc := make([]int, m)
	for i := range asc {
		asc[i] = i
	}
	_, fc := quickselect(asc, m/2, nil, false)
	// every pivot is the minimum of its range, so each round removes ONE
	// element: rounds k+1, comparisons (m-1) + (m-2) + ... + (m-1-k).
	k := m / 2
	fmt.Printf("  n = %d: %d comparisons = %.0fx n; formula (k+1)(2n-2-k)/2 = %d\n",
		m, fc, float64(fc)/m, (k+1)*(2*m-2-k)/2)

	fmt.Println("\n5) the standard library on the same permutation")
	// Go has no selection function: the stdlib route is sort, then index. Its
	// cost is the same whichever k you want.
	work = slices.Clone(xs)
	sc := 0
	slices.SortFunc(work, func(a, b int) int { sc++; return cmp.Compare(a, b) })
	fmt.Printf("  slices.SortFunc + index: %d comparisons, any k\n", sc)
	fmt.Println("        k   quickselect   = x n")
	for _, k := range []int{0, n / 10, n / 4, n / 2, n - 1} {
		work = slices.Clone(xs)
		v, q := quickselect(work, k, r, true)
		if v != work[k] || v != k {
			panic("wrong answer")
		}
		fmt.Printf("  %7d   %11d   %5.2f\n", k, q, float64(q)/n)
	}
}
```

**Output:**

```
1) random-pivot quickselect, n = 100000, average of 200 permutations
        k   comparisons/n   +-2 s.e.   theory   theory inside?
        0           2.047      0.104    2.000   true
    10000           2.639      0.122    2.650   true
    25000           3.117      0.128    3.125   true
    50000           3.460      0.137    3.386   true

2) the median of one permutation: sort-and-index vs quickselect
  merge sort, then index :   1536292 comparisons -> 50000
  quickselect            :    261541 comparisons -> 50000
  quickselect does 5.9x fewer comparisons

3) top-10: select k = 9, then sort only the 10 on the left
  [0 1 2 3 4 5 6 7 8 9]  select 223777 + sort 19 = 223796 comparisons
  equal to the full sort's first 10: true (full sort: 1536292 comparisons)

4) first-element pivot, already-sorted input, median
  n = 10000: 37502499 comparisons = 3750x n; formula (k+1)(2n-2-k)/2 = 37502499

5) the standard library on the same permutation
  slices.SortFunc + index: 1761852 comparisons, any k
        k   quickselect   = x n
        0        230231    2.30
    10000        216861    2.17
    25000        315537    3.16
    50000        237899    2.38
    99999        215614    2.16
```

**Rust:**

```rust
// Quickselect finds the k-th smallest element (0-based) without sorting:
// partition once, then continue into ONLY the side that holds index k. The
// other side is never looked at again, so the work is n + n/2 + n/4 + ... on
// average -- linear -- instead of quicksort's n per level for lg n levels.

/// PCG-DXSM, a port of Go's math/rand/v2 PCG: same seeds, same stream, so the
/// permutations and the random pivots are exactly the Go program's.
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

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
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

/// Lomuto's scheme on xs[lo..=hi] with the pivot already at hi. It makes
/// exactly (hi - lo) comparisons -- one per non-pivot element -- and returns
/// the pivot's final index: everything left of it is smaller.
fn partition(xs: &mut [i64], lo: usize, hi: usize, cmps: &mut u64) -> usize {
    let p = xs[hi];
    let mut i = lo;
    for j in lo..hi {
        *cmps += 1;
        if xs[j] < p {
            xs.swap(i, j);
            i += 1;
        }
    }
    xs.swap(i, hi);
    i
}

/// Iterative (a loop, no recursion: O(1) extra space). With `Some(rng)` it
/// swaps a random element of the range to the end first; with `None` it uses
/// the FIRST element -- the textbook rule that sorted input defeats.
/// Returns (the k-th smallest, comparisons).
fn quickselect(xs: &mut [i64], k: usize, mut rng: Option<&mut Pcg>) -> (i64, u64) {
    let mut cmps = 0;
    let (mut lo, mut hi) = (0, xs.len() - 1);
    while lo < hi {
        let pv = match rng.as_deref_mut() {
            Some(r) => lo + r.int_n(hi - lo + 1),
            None => lo,
        };
        xs.swap(pv, hi);
        let m = partition(xs, lo, hi, &mut cmps);
        // usize indexes: `hi = m - 1` would underflow at m = 0, but k < m
        // implies m >= 1, so it cannot happen here.
        match k.cmp(&m) {
            std::cmp::Ordering::Less => hi = m - 1, // k is left; drop the right part for good
            std::cmp::Ordering::Greater => lo = m + 1, // k is in the right part
            std::cmp::Ordering::Equal => return (xs[k], cmps), // the pivot landed on k
        }
    }
    (xs[k], cmps)
}

/// Top-down merge sort, one scratch buffer: the "sort, then index" baseline.
fn merge_sort(xs: &mut [i64], buf: &mut [i64], cmps: &mut u64) {
    if xs.len() < 2 {
        return;
    }
    let mid = xs.len() / 2;
    {
        let (bl, br) = buf.split_at_mut(mid);
        merge_sort(&mut xs[..mid], bl, cmps);
        merge_sort(&mut xs[mid..], br, cmps);
    }
    let (mut i, mut j, mut o) = (0, mid, 0);
    while i < mid && j < xs.len() {
        *cmps += 1;
        if xs[j] < xs[i] {
            buf[o] = xs[j];
            j += 1;
        } else {
            buf[o] = xs[i];
            i += 1;
        }
        o += 1;
    }
    let left = mid - i;
    buf[o..o + left].copy_from_slice(&xs[i..mid]);
    o += left;
    buf[o..xs.len()].copy_from_slice(&xs[j..]);
    let n = xs.len();
    xs.copy_from_slice(&buf[..n]);
}

/// Fisher-Yates permutation of 0..n-1 driven by int_n only -- the same calls,
/// in the same order, as the Go program.
fn shuffled(n: usize, r: &mut Pcg) -> Vec<i64> {
    let mut xs: Vec<i64> = (0..n as i64).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i + 1);
        xs.swap(i, j);
    }
    xs
}

/// Knuth's expected comparisons per element for random-pivot quickselect, to
/// leading order: 2 + 2a ln(1/a) + 2(1-a) ln(1/(1-a)), a = k/n.
fn theory(k: usize, n: usize) -> f64 {
    let a = k as f64 / n as f64;
    let mut t = 2.0;
    if a > 0.0 {
        t += 2.0 * a * (1.0 / a).ln();
    }
    if a < 1.0 {
        t += 2.0 * (1.0 - a) * (1.0 / (1.0 - a)).ln();
    }
    t
}

fn main() {
    let mut r = Pcg::new(1, 2);
    const N: usize = 100_000;
    const TRIALS: usize = 200;
    let nf = N as f64;
    let ks = [0, N / 10, N / 4, N / 2];

    println!("1) random-pivot quickselect, n = {N}, average of {TRIALS} permutations");
    let mut sums = [0.0f64; 4]; // sum of comparisons/n
    let mut sqs = [0.0f64; 4]; // sum of squares, for the spread
    let mut work = vec![0i64; N];
    for _ in 0..TRIALS {
        let xs = shuffled(N, &mut r);
        for (i, &k) in ks.iter().enumerate() {
            work.copy_from_slice(&xs);
            let (v, c) = quickselect(&mut work, k, Some(&mut r));
            assert_eq!(
                v, k as i64,
                "a permutation of 0..n-1: the k-th smallest IS k"
            );
            let x = c as f64 / nf;
            sums[i] += x;
            sqs[i] += x * x;
        }
    }
    // One run varies a lot (its standard deviation is about n comparisons), so
    // print the mean with +-2 standard errors and check the theory against that.
    println!("        k   comparisons/n   +-2 s.e.   theory   theory inside?");
    let t = TRIALS as f64;
    for (i, &k) in ks.iter().enumerate() {
        let mean = sums[i] / t;
        let se = ((sqs[i] / t - mean * mean) / t).sqrt();
        let th = theory(k, N);
        println!(
            "  {k:>7}   {mean:>13.3}   {:>8.3}   {th:>6.3}   {}",
            2.0 * se,
            (mean - th).abs() <= 2.0 * se
        );
    }

    println!("\n2) the median of one permutation: sort-and-index vs quickselect");
    let xs = shuffled(N, &mut r);
    let mut sorted = xs.clone();
    let mut ms = 0;
    merge_sort(&mut sorted, &mut vec![0; N], &mut ms);
    let mut work = xs.clone();
    let (med, qs) = quickselect(&mut work, N / 2, Some(&mut r));
    println!(
        "  merge sort, then index : {ms:>9} comparisons -> {}",
        sorted[N / 2]
    );
    println!("  quickselect            : {qs:>9} comparisons -> {med}");
    println!(
        "  quickselect does {:.1}x fewer comparisons",
        ms as f64 / qs as f64
    );

    println!("\n3) top-10: select k = 9, then sort only the 10 on the left");
    let mut work = xs.clone();
    let (_, tc) = quickselect(&mut work, 9, Some(&mut r));
    let top = &mut work[..10]; // left of k: all <= xs[9], in no particular order
    let mut ins = 0; // insertion sort on just these 10
    for i in 1..top.len() {
        for j in (1..=i).rev() {
            ins += 1;
            if top[j] >= top[j - 1] {
                break;
            }
            top.swap(j, j - 1);
        }
    }
    let shown: Vec<String> = top.iter().map(i64::to_string).collect();
    println!(
        "  [{}]  select {tc} + sort {ins} = {} comparisons",
        shown.join(" "),
        tc + ins
    );
    println!(
        "  equal to the full sort's first 10: {} (full sort: {ms} comparisons)",
        *top == sorted[..10]
    );

    println!("\n4) first-element pivot, already-sorted input, median");
    const M: usize = 10_000;
    let mut asc: Vec<i64> = (0..M as i64).collect();
    let (_, fc) = quickselect(&mut asc, M / 2, None);
    // every pivot is the minimum of its range, so each round removes ONE
    // element: rounds k+1, comparisons (m-1) + (m-2) + ... + (m-1-k).
    let k = M / 2;
    println!(
        "  n = {M}: {fc} comparisons = {:.0}x n; formula (k+1)(2n-2-k)/2 = {}",
        fc as f64 / M as f64,
        (k + 1) * (2 * M - 2 - k) / 2
    );

    println!("\n5) the standard library on the same permutation");
    // Rust's stdlib route for "sort, then index" -- for parity with Go's line.
    let mut work = xs.clone();
    let mut sc = 0u64;
    work.sort_unstable_by(|a, b| {
        sc += 1;
        a.cmp(b)
    });
    println!("  sort_unstable_by + index: {sc} comparisons, any k");
    // And the real thing: select_nth_unstable_by, counted through its closure.
    // It returns (left, &mut kth, right) with left <= kth <= right.
    println!("        k   quickselect   = x n   select_nth_unstable_by   = x n");
    for k in [0, N / 10, N / 4, N / 2, N - 1] {
        let mut work = xs.clone();
        let (v, q) = quickselect(&mut work, k, Some(&mut r));
        assert!(v == work[k] && v == k as i64, "wrong answer");
        let mut work = xs.clone();
        let mut s = 0u64;
        let (left, kth, right) = work.select_nth_unstable_by(k, |a, b| {
            s += 1;
            a.cmp(b)
        });
        assert!(*kth == k as i64 && left.iter().all(|x| x < kth) && right.iter().all(|x| x > kth));
        println!(
            "  {k:>7}   {q:>11}   {:>5.2}   {s:>22}   {:>5.2}",
            q as f64 / nf,
            s as f64 / nf
        );
    }
}
```

**Rust output:**

```
1) random-pivot quickselect, n = 100000, average of 200 permutations
        k   comparisons/n   +-2 s.e.   theory   theory inside?
        0           2.047      0.104    2.000   true
    10000           2.639      0.122    2.650   true
    25000           3.117      0.128    3.125   true
    50000           3.460      0.137    3.386   true

2) the median of one permutation: sort-and-index vs quickselect
  merge sort, then index :   1536292 comparisons -> 50000
  quickselect            :    261541 comparisons -> 50000
  quickselect does 5.9x fewer comparisons

3) top-10: select k = 9, then sort only the 10 on the left
  [0 1 2 3 4 5 6 7 8 9]  select 223777 + sort 19 = 223796 comparisons
  equal to the full sort's first 10: true (full sort: 1536292 comparisons)

4) first-element pivot, already-sorted input, median
  n = 10000: 37502499 comparisons = 3750x n; formula (k+1)(2n-2-k)/2 = 37502499

5) the standard library on the same permutation
  sort_unstable_by + index: 1720422 comparisons, any k
        k   quickselect   = x n   select_nth_unstable_by   = x n
        0        230231    2.30                    99999    1.00
    10000        216861    2.17                   223336    2.23
    25000        315537    3.16                   212624    2.13
    50000        237899    2.38                   195411    1.95
    99999        215614    2.16                    99999    1.00
```

> **Go vs Rust:** Sections 1–4 are identical to Go (same PCG stream, same Lomuto partition). Section 5 differs by design: Go has no selection function, so its line is `slices.SortFunc` + index (1,761,852; Rust `sort_unstable_by` + index makes 1,720,422 — a different algorithm), while Rust's `select_nth_unstable_by` made exactly n − 1 = 99,999 comparisons for k = 0 and k = n − 1 (`partition_at_index` in `core/src/slice/sort/select.rs` does a linear min/max scan there) and 195,411 (1.95n) for the median.
> Its pivot is the recursive median-of-3 that ipnsort uses (`shared/pivot.rs`) — the plausible reason it beats our random pivot's 237,899 on this input, not measured — and the median-of-medians fallback after 16 bad rounds was read in the source, not run.

**Complexity:** expected Θ(n): measured 2.05n comparisons for the minimum and 3.46n for the median over 200 permutations (theory 2.00n and 3.39n, both inside ±2 standard errors — one run's spread is about n) · worst case Θ(n²): a first-element pivot on sorted input made exactly (k+1)(2n−2−k)/2 = 37,502,499 comparisons at n = 10,000 · O(1) extra space (iterative) · for the median, 5.9× fewer comparisons than merge sort + index and 7.4× fewer than `slices.SortFunc` + index (1,761,852).

---

## 11. What `slices.Sort` actually runs

`🟡 medium` · *pdqsort / reading the standard library*

You cannot put a print statement inside `slices.SortFunc`, but you can give it a comparator that counts its
own calls, and then build inputs that each reach a single branch of the source. Input that is already in
order costs a check of n − 1 comparisons plus a fixed amount, and that fixed amount is what the pivot sample
cost. Swapping a few adjacent pairs shows exactly how many the "probably sorted" repair will fix before it
gives up. Reversed input shows that the reverse trick only works when the ninther took the sample. Each count
is then matched to a named piece of `src/slices/zsortanyfunc.go` — and checked in the program, which stops if
a future Go release changes those rules. The insertion-sort cutoff was already caught in lesson 12, example 12,
so every n here is above it. The point is the rest of pdqsort: Θ(n) on sorted, reversed and all-equal input,
Θ(n log n) elsewhere, with a heapsort fallback for the worst case.

**Steps:**

1. Write `count(sortFn, xs)`: it runs `slices.SortFunc` or `slices.SortStableFunc` with a closure that adds one
   to a counter and then calls `cmp.Compare`.
2. Feed it sorted, reversed and all-equal input around n = 50, then sorted input with k = 0…6 adjacent pairs
   swapped far apart, then six shapes at n = 100,000, and measure allocations with `testing.AllocsPerRun`.
3. Read each count against the source: +3 vs +12 is median of 3 vs the ninther, reversed input is handled in
   one pass only from n = 50, and the repair fixes at most 4 pairs, not 5.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
	"strings"
	"testing"
)

// slices.SortFunc is pdqsort; slices.SortStableFunc is insertion-sorted blocks
// plus SymMerge. We cannot put a print statement inside the standard library,
// but we can hand it a comparator that counts its own calls -- and on inputs
// crafted to hit one branch of the source at a time, the count says which
// branch ran. (Lesson 12, example 12 already caught the insertion-sort cutoff
// this way, so every n here is above it.)

// count sorts xs with sortFn and returns how many times it called cmp.
func count(sortFn func([]int, func(a, b int) int), xs []int) int {
	calls := 0
	sortFn(xs, func(a, b int) int { calls++; return cmp.Compare(a, b) })
	return calls
}

var (
	unstable = slices.SortFunc[[]int, int]
	stable   = slices.SortStableFunc[[]int, int]
)

// The input shapes, built the same way in the Rust twin.
func ascending(n int) []int {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = i
	}
	return xs
}

func descending(n int) []int {
	xs := make([]int, n)
	for i := range xs {
		xs[i] = n - 1 - i
	}
	return xs
}

func main() {
	fmt.Println("1) input already in order: comparisons, and the extra over the n-1 a check costs")
	fmt.Println("       n   sorted  reversed  all-equal   extra: sorted reversed equal")
	var shortReversed []string // extras of reversed inputs that were NOT handled in one pass
	for _, n := range []int{21, 49, 50, 1_000, 100_000} {
		s := count(unstable, ascending(n))
		r := count(unstable, descending(n))
		e := count(unstable, make([]int, n))
		fmt.Printf("  %6d  %7d  %8d  %9d   %+13d %+8d %+5d\n", n, s, r, e, s-(n-1), r-(n-1), e-(n-1))
		// The source map in section 4 is checked here, not just asserted:
		// a future Go version that changes the pivot rules stops this program.
		want := 3 // median of 3 below shortestNinther = 50
		if n >= 50 {
			want = 12 // Tukey's ninther
		}
		if s-(n-1) != want || e != s || (n >= 50) != (r == s) {
			panic("the source map below no longer matches this Go version")
		}
		if r != s {
			shortReversed = append(shortReversed, fmt.Sprintf("%+d at n = %d", r-(n-1), n))
		}
	}

	const n = 100_000
	fmt.Printf("\n2) sorted n = %d with k adjacent pairs swapped, 16,000 apart\n", n)
	fmt.Println("  k  SortFunc  extra over k=0  SortStableFunc")
	var counts []int
	for k := 0; k <= 6; k++ {
		xs := ascending(n)
		for j := 0; j < k; j++ { // positions avoid the pivot sample at n/4, n/2, 3n/4
			p := 1_000 + 16_000*j
			xs[p], xs[p+1] = xs[p+1], xs[p]
		}
		u := count(unstable, slices.Clone(xs))
		st := count(stable, xs)
		counts = append(counts, u)
		fmt.Printf("  %d  %8d  %+14d  %14d\n", k, u, u-counts[0], st)
	}
	cheap := 0 // largest k still costing one pass plus 3 comparisons per pair
	for cheap+1 < len(counts) && counts[cheap+1]-counts[0] == 3*(cheap+1) {
		cheap++
	}
	if cheap != 4 { // maxSteps - 1 in partialInsertionSort
		panic("the source map below no longer matches this Go version")
	}
	fmt.Printf("  up to k = %d: one pass plus 3 comparisons per pair; at k = %d: %.2fx the k = 0 cost\n",
		cheap, cheap+1, float64(counts[cheap+1])/float64(counts[0]))

	fmt.Printf("\n3) six shapes, n = %d: comparisons and comparisons / (n lg n)\n", n)
	r := rand.New(rand.NewPCG(1, 2))
	random := ascending(n)
	for i := n - 1; i > 0; i-- { // Fisher-Yates over IntN, as in the Rust twin
		j := r.IntN(i + 1)
		random[i], random[j] = random[j], random[i]
	}
	organ, saw := make([]int, n), make([]int, n)
	for i := range n {
		organ[i] = min(i, n-1-i) // 0,1,...,n/2-1, n/2-1,...,0
		saw[i] = i % 64
	}
	nlgn := n * math.Log2(n)
	fmt.Println("  shape        SortFunc  ratio   SortStableFunc  ratio")
	for _, sh := range []struct {
		name string
		xs   []int
	}{
		{"sorted", ascending(n)}, {"reversed", descending(n)}, {"all-equal", make([]int, n)},
		{"organ-pipe", organ}, {"sawtooth", saw}, {"random", random},
	} {
		u := count(unstable, slices.Clone(sh.xs))
		s := count(stable, slices.Clone(sh.xs))
		fmt.Printf("  %-10s  %9d  %5.3f  %15d  %5.3f\n", sh.name, u, float64(u)/nlgn, s, float64(s)/nlgn)
	}
	w := make([]int, n)
	byValue := func(a, b int) int { return cmp.Compare(a, b) }
	au := testing.AllocsPerRun(5, func() { copy(w, random); slices.SortFunc(w, byValue) })
	as := testing.AllocsPerRun(5, func() { copy(w, random); slices.SortStableFunc(w, byValue) })
	fmt.Printf("  heap allocations per call on random: SortFunc %.0f, SortStableFunc %.0f\n", au, as)

	fmt.Printf(`
4) the counts, mapped to $(go env GOROOT)/src/slices/zsortanyfunc.go
   (SortFunc's copy of the pdqsort in zsortordered.go, which slices.Sort uses)
  +3 below n = 50       choosePivot: median of 3 indices for 8 <= length < 50 = 3 cmps
  +12 from n = 50       choosePivot: Tukey's ninther (shortestNinther = 50), 4 medians = 12 cmps;
                        it counts swaps: 0 -> increasingHint, all 12 (maxSwaps) -> decreasingHint
  the n-1               partialInsertionSort: one pass; returns true when it reaches the end
  reversed, n >= 50     decreasingHint -> reverseRange (0 cmps), then the same pass as sorted;
                        median of 3 counts at most 3 swaps, so a shorter reversed range is
                        never reversed and runs the full quicksort (%s)
  k <= 4 cheap, 5 not   maxSteps = 5 steps, each scans to the next bad pair and fixes it
                        (+3 cmps); only a scan that reaches the end returns true, so 5 bad
                        pairs use up every step and it returns false -- the input is sorted
                        by then, but the partitioning runs anyway
  all-equal = sorted    nothing is less, so no sample swaps and the pass succeeds:
                        partitionEqual, the duplicate trick, is never reached
  heapsort fallback     limit = bits.Len(n) = %d bad splits at n = %d, then heapSort the range;
                        read in the source, not forced here -- example 12 builds that input
  SortStableFunc        insertion sort on blocks of 20, then symMerge rounds: no run
                        detection, so sorted input still pays for every merge
`, strings.Join(shortReversed, ", "), bits.Len(uint(n)), n)
}
```

**Output:**

```
1) input already in order: comparisons, and the extra over the n-1 a check costs
       n   sorted  reversed  all-equal   extra: sorted reversed equal
      21       23        57         23              +3      +37    +3
      49       51       187         51              +3     +139    +3
      50       61        61         61             +12      +12   +12
    1000     1011      1011       1011             +12      +12   +12
  100000   100011    100011     100011             +12      +12   +12

2) sorted n = 100000 with k adjacent pairs swapped, 16,000 apart
  k  SortFunc  extra over k=0  SortStableFunc
  0    100011              +0          120210
  1    100014              +3          120210
  2    100017              +6          120210
  3    100020              +9          120210
  4    100023             +12          120210
  5    265048         +165037          120210
  6    265051         +165040          120210
  up to k = 4: one pass plus 3 comparisons per pair; at k = 5: 2.65x the k = 0 cost

3) six shapes, n = 100000: comparisons and comparisons / (n lg n)
  shape        SortFunc  ratio   SortStableFunc  ratio
  sorted         100011  0.060           120210  0.072
  reversed       100011  0.060           980261  0.590
  all-equal      100011  0.060           120210  0.072
  organ-pipe    1617976  0.974           716535  0.431
  sawtooth       737314  0.444           715943  0.431
  random        1756169  1.057          2098212  1.263
  heap allocations per call on random: SortFunc 0, SortStableFunc 0

4) the counts, mapped to $(go env GOROOT)/src/slices/zsortanyfunc.go
   (SortFunc's copy of the pdqsort in zsortordered.go, which slices.Sort uses)
  +3 below n = 50       choosePivot: median of 3 indices for 8 <= length < 50 = 3 cmps
  +12 from n = 50       choosePivot: Tukey's ninther (shortestNinther = 50), 4 medians = 12 cmps;
                        it counts swaps: 0 -> increasingHint, all 12 (maxSwaps) -> decreasingHint
  the n-1               partialInsertionSort: one pass; returns true when it reaches the end
  reversed, n >= 50     decreasingHint -> reverseRange (0 cmps), then the same pass as sorted;
                        median of 3 counts at most 3 swaps, so a shorter reversed range is
                        never reversed and runs the full quicksort (+37 at n = 21, +139 at n = 49)
  k <= 4 cheap, 5 not   maxSteps = 5 steps, each scans to the next bad pair and fixes it
                        (+3 cmps); only a scan that reaches the end returns true, so 5 bad
                        pairs use up every step and it returns false -- the input is sorted
                        by then, but the partitioning runs anyway
  all-equal = sorted    nothing is less, so no sample swaps and the pass succeeds:
                        partitionEqual, the duplicate trick, is never reached
  heapsort fallback     limit = bits.Len(n) = 17 bad splits at n = 100000, then heapSort the range;
                        read in the source, not forced here -- example 12 builds that input
  SortStableFunc        insertion sort on blocks of 20, then symMerge rounds: no run
                        detection, so sorted input still pays for every merge
```

**Rust:**

```rust
// `sort_unstable_by` is ipnsort; `sort_by` (the STABLE one, with the short
// name) is driftsort. We cannot put a print statement inside the standard
// library, but we can hand it a comparator that counts its own calls -- and on
// inputs crafted to hit one branch of the source at a time, the count says
// which branch ran. (Lesson 12, example 12 already caught the insertion-sort
// cutoff this way, so every n here is above it.)

use std::alloc::{GlobalAlloc, Layout, System};
use std::cmp::Ordering;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// ---- allocation counter: Rust's stand-in for testing.AllocsPerRun ----
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0); // allocation events
static TOTAL: AtomicUsize = AtomicUsize::new(0); // bytes requested

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        TOTAL.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
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

/// PCG-DXSM, a port of Go's math/rand/v2 PCG: same seeds, same permutation.
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

    /// like rand.IntN(n): exactly uniform in [0, n), Lemire's method with rejection.
    fn int_n(&mut self, n: usize) -> usize {
        let n = n as u64;
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

/// Which standard-library sort to run. Rust's two sorts are methods, not free
/// functions, so an enum + match stands in for Go's function values.
#[derive(Clone, Copy)]
enum Sort {
    Unstable, // sort_unstable_by: ipnsort
    Stable,   // sort_by: driftsort
}

/// Sorts xs and returns how many times the comparator was called.
fn count(which: Sort, xs: &mut [i64]) -> usize {
    let mut calls = 0;
    let mut cmp = |a: &i64, b: &i64| -> Ordering {
        calls += 1;
        a.cmp(b)
    };
    match which {
        Sort::Unstable => xs.sort_unstable_by(&mut cmp),
        Sort::Stable => xs.sort_by(&mut cmp),
    }
    calls
}

// The input shapes, built the same way as in the Go program.
fn ascending(n: usize) -> Vec<i64> {
    (0..n as i64).collect()
}

fn descending(n: usize) -> Vec<i64> {
    (0..n as i64).rev().collect()
}

/// sorted, then k adjacent pairs swapped, 16,000 apart.
fn with_swaps(n: usize, k: usize) -> Vec<i64> {
    let mut xs = ascending(n);
    for j in 0..k {
        let p = 1_000 + 16_000 * j;
        xs.swap(p, p + 1);
    }
    xs
}

fn main() {
    use Sort::{Stable, Unstable};

    println!("1) input already in order: comparisons, and the extra over the n-1 a check costs");
    println!("       n   sorted  reversed  all-equal   extra: sorted reversed equal");
    for n in [21, 49, 50, 1_000, 100_000] {
        let s = count(Unstable, &mut ascending(n));
        let r = count(Unstable, &mut descending(n));
        let e = count(Unstable, &mut vec![0; n]);
        let x = |c: usize| c as i64 - (n as i64 - 1);
        println!(
            "  {n:>6}  {s:>7}  {r:>8}  {e:>9}   {:>+13} {:>+8} {:>+5}",
            x(s),
            x(r),
            x(e)
        );
        // The source map in section 4 is checked here, not just asserted.
        assert!(
            s == n - 1 && r == n - 1 && e == n - 1,
            "the source map below no longer matches this Rust version"
        );
    }

    const N: usize = 100_000;
    println!("\n2) sorted n = {N} with k adjacent pairs swapped, 16,000 apart");
    println!("  k  sort_unstable_by  extra over k=0  sort_by");
    let mut counts = Vec::new();
    for k in 0..=6 {
        let u = count(Unstable, &mut with_swaps(N, k));
        let st = count(Stable, &mut with_swaps(N, k));
        counts.push(u);
        println!(
            "  {k}  {u:>16}  {:>+14}  {st:>7}",
            u as i64 - counts[0] as i64
        );
    }
    assert!(
        counts[1] > 10 * counts[0],
        "the source map below no longer matches this Rust version"
    );
    println!(
        "  already at k = 1: {:.2}x the k = 0 cost -- no repair step, a full quicksort",
        counts[1] as f64 / counts[0] as f64
    );

    println!("\n3) six shapes, n = {N}: comparisons and comparisons / (n lg n)");
    let mut r = Pcg::new(1, 2);
    let mut random = ascending(N);
    for i in (1..N).rev() {
        // Fisher-Yates over int_n, as in the Go program
        let j = r.int_n(i + 1);
        random.swap(i, j);
    }
    let organ: Vec<i64> = (0..N).map(|i| i.min(N - 1 - i) as i64).collect();
    let saw: Vec<i64> = (0..N).map(|i| (i % 64) as i64).collect();
    let nlgn = N as f64 * (N as f64).log2();
    println!("  shape       sort_unstable_by  ratio   sort_by  ratio");
    let shapes = [
        ("sorted", ascending(N)),
        ("reversed", descending(N)),
        ("all-equal", vec![0; N]),
        ("organ-pipe", organ),
        ("sawtooth", saw),
        ("random", random.clone()),
    ];
    for (name, xs) in &shapes {
        let u = count(Unstable, &mut xs.clone());
        let s = count(Stable, &mut xs.clone());
        println!(
            "  {name:<10}  {u:>16}  {:>5.3}  {s:>8}  {:>5.3}",
            u as f64 / nlgn,
            s as f64 / nlgn
        );
    }
    // Allocations: the counting allocator sees every one, deterministically.
    // The copy target is allocated outside the measured closures.
    let mut w = vec![0i64; N];
    for (name, src) in [("random", &random), ("sorted", &shapes[0].1)] {
        let (ua, ub) = allocs_of(|| {
            w.copy_from_slice(src);
            w.sort_unstable();
        });
        let (sa, sb) = allocs_of(|| {
            w.copy_from_slice(src);
            w.sort();
        });
        println!(
            "  heap allocations per call on {name}: sort_unstable {ua} ({ub} bytes), sort {sa} ({sb} bytes)"
        );
    }

    println!(
        "
4) the counts, mapped to $(rustc --print sysroot)/lib/rustlib/src/rust/library/core/src/slice/sort/
  +0 everywhere         unstable/mod.rs ipnsort: find_existing_run scans the WHOLE slice first;
                        one ascending (non-descending) or strictly descending run = n-1 cmps,
                        reverse if descending, done. No pivot sample is taken at all
  all-equal = sorted    equal neighbours are 'not less', so it is one non-descending run
  k >= 1: full sort     the scan stops at the first bad pair and quicksort(.., limit) sorts
                        everything; Rust's quicksort has no partialInsertionSort-style repair
  pivot                 shared/pivot.rs choose_pivot: median of 3 below 64, recursive
                        median of 3 (pseudo-median of ~n^0.53 samples) from 64
  heapsort fallback     limit = 2 * ilog2(n) = {} at n = {N}, spent on EVERY partition level;
                        read in the source, not forced here -- example 12 builds that input
  duplicates            quicksort: if the ancestor pivot is not less than the new pivot,
                        partition into <= and > and drop the equal block
  sort_by (driftsort)   stable/drift.rs: natural runs of >= sqrt(n) (n > 4096) are kept,
                        shorter stretches are sorted by a stable quicksort, then the runs
                        are merged (powersort order). stable/mod.rs allocates
                        max(n - n/2, min(n, 8MB/size)) elements BEFORE looking at the data,
                        so sorted input allocates as much as random (above)",
        2 * (N | 1).ilog2()
    );
}
```

**Rust output:**

```
1) input already in order: comparisons, and the extra over the n-1 a check costs
       n   sorted  reversed  all-equal   extra: sorted reversed equal
      21       20        20         20              +0       +0    +0
      49       48        48         48              +0       +0    +0
      50       49        49         49              +0       +0    +0
    1000      999       999        999              +0       +0    +0
  100000    99999     99999      99999              +0       +0    +0

2) sorted n = 100000 with k adjacent pairs swapped, 16,000 apart
  k  sort_unstable_by  extra over k=0  sort_by
  0             99999              +0    99999
  1           1686061        +1586062   101001
  2           1686062        +1586063   118003
  3           1686062        +1586063   150004
  4           1686062        +1586063   183006
  5           1686061        +1586062   199007
  6           1686061        +1586062   218007
  already at k = 1: 16.86x the k = 0 cost -- no repair step, a full quicksort

3) six shapes, n = 100000: comparisons and comparisons / (n lg n)
  shape       sort_unstable_by  ratio   sort_by  ratio
  sorted                 99999  0.060     99999  0.060
  reversed               99999  0.060     99999  0.060
  all-equal              99999  0.060     99999  0.060
  organ-pipe           2149584  1.294    199998  0.120
  sawtooth              730318  0.440    811432  0.489
  random               1721445  1.036   1744466  1.050
  heap allocations per call on random: sort_unstable 0 (0 bytes), sort 1 (800000 bytes)
  heap allocations per call on sorted: sort_unstable 0 (0 bytes), sort 1 (800000 bytes)

4) the counts, mapped to $(rustc --print sysroot)/lib/rustlib/src/rust/library/core/src/slice/sort/
  +0 everywhere         unstable/mod.rs ipnsort: find_existing_run scans the WHOLE slice first;
                        one ascending (non-descending) or strictly descending run = n-1 cmps,
                        reverse if descending, done. No pivot sample is taken at all
  all-equal = sorted    equal neighbours are 'not less', so it is one non-descending run
  k >= 1: full sort     the scan stops at the first bad pair and quicksort(.., limit) sorts
                        everything; Rust's quicksort has no partialInsertionSort-style repair
  pivot                 shared/pivot.rs choose_pivot: median of 3 below 64, recursive
                        median of 3 (pseudo-median of ~n^0.53 samples) from 64
  heapsort fallback     limit = 2 * ilog2(n) = 32 at n = 100000, spent on EVERY partition level;
                        read in the source, not forced here -- example 12 builds that input
  duplicates            quicksort: if the ancestor pivot is not less than the new pivot,
                        partition into <= and > and drop the equal block
  sort_by (driftsort)   stable/drift.rs: natural runs of >= sqrt(n) (n > 4096) are kept,
                        shorter stretches are sorted by a stable quicksort, then the runs
                        are merged (powersort order). stable/mod.rs allocates
                        max(n - n/2, min(n, 8MB/size)) elements BEFORE looking at the data,
                        so sorted input allocates as much as random (above)
```

> **Go vs Rust:** Same inputs (and the same PCG permutation), different algorithms, so every count differs by design. ipnsort's `find_existing_run` scans the whole slice first: sorted, reversed and all-equal input cost exactly n − 1 at every n (no pivot sample, so +0 where Go pays +3 or +12, and no n = 50 threshold for reversed input). But a single swapped pair sends it into the full quicksort — 1,686,061 comparisons, 16.86× the sorted cost and close to random's 1,721,445 — where Go's `partialInsertionSort` repairs up to 4 pairs for 3 comparisons each.
> `sort_by` (driftsort) adapts where Go's `SortStableFunc` cannot (99,999 on sorted and reversed input, 199,998 on organ-pipe against Go's 716,535), and it pays with one 800,000-byte allocation per call, even on sorted input, where both Go sorts allocate nothing.

**Complexity:** pdqsort is Θ(n) on sorted, reversed and all-equal input (n − 1 + 12 = 100,011 comparisons at n = 100,000) and on sorted input with up to 4 adjacent pairs out of place (+3 comparisons each); a 5th pair costs 2.65× as much (265,048) · Θ(n log n) elsewhere (random 1.057 n lg n, organ-pipe 0.974), and Θ(n log n) in the worst case through the heapsort fallback after `bits.Len(n)` = 17 bad splits (read in the source, not forced here) · `SortStableFunc` has no run detection — 120,210 comparisons on sorted input, 980,261 on reversed — and, like `SortFunc`, makes 0 allocations.

---

> Next tier: [🔴 hard](3-hard.md) — the adversary that breaks any fixed quicksort, the hybrid that
> survives it, sorting more data than fits in memory, the full race, and the capstone.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
