# Step 20 — Heaps & Priority Queues · 🟡 Medium

Examples **7–11**: top-K in Θ(n log k), merging k sorted lists four ways, the running median with two
heaps, meeting rooms and a cooldown scheduler, and lazy deletion with the memory it costs.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic and should match character for character; **Sample
output:** ones print timings.

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

## 7. Top-K in Θ(n log k)

`🟡 medium` · *Bounded heap / streaming*

To keep the k largest values of a stream, hold a **min**-heap of the k best so far: its root is the smallest of them, the bar a newcomer must clear. Each element costs one comparison with the root, and only an element that clears the bar replaces it and sinks back into place, so the total is Θ(n log k) time and Θ(k) space in a single pass over input you never store. On random order an element beats the bar with probability k/i, so the number of replacements is about k ln(n/k) and the real cost sits far below the bound; the worst case is input that beats the root every time. Quickselect ([lesson 13](../../13-efficient-sorts.md)) is Θ(n) on average but needs the whole input in memory, and top-K *frequent* ([lesson 14](../../14-linear-sorts.md), example 11) counts first and then picks.

**Steps:**

1. Write a min-heap of ints with `up`/`down` loops copied from `container/heap` and a counter inside `less`, so each count is exactly the number of comparisons.
2. Stream n = 100,000 distinct values in three orders through two ways of replacing the root (overwrite and sift down, or pop then push) for k = 10, 100, 1000, 10000.
3. Compare the counts with k ln(n/k) and n lg k, average the replacements over 100 permutations, then check the heap against `container/heap`, against sorting, and against 5,000 brute-force cases with ties.

```go
package main

import (
	"cmp"
	"container/heap"
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
)

// Top-K of a stream: keep a MIN-heap of the k largest values seen so far.
// The root is the smallest of the k, i.e. the bar a newcomer must clear. Each
// element costs one comparison with the root; only if it clears the bar does it
// replace the root and sink back into place: Theta(n log k) time, Theta(k) space,
// one pass, and the stream is never stored.

// minHeap is a min-heap of ints whose loops are copied from container/heap's up
// and down (strict <, left child on ties, a swap at every level), and which
// counts every call of less. The Rust twin ports the same loops, so every count
// below is identical in both languages.
type minHeap struct {
	a    []int
	cmps int
}

func (h *minHeap) less(i, j int) bool { h.cmps++; return h.a[i] < h.a[j] }
func (h *minHeap) swap(i, j int)      { h.a[i], h.a[j] = h.a[j], h.a[i] }

func (h *minHeap) up(j int) {
	for {
		i := (j - 1) / 2 // parent; at j == 0 Go's division gives 0, hence the i == j test
		if i == j || !h.less(j, i) {
			break
		}
		h.swap(i, j)
		j = i
	}
}

func (h *minHeap) down(i, n int) {
	for {
		j1 := 2*i + 1
		if j1 >= n {
			break
		}
		j := j1
		if j2 := j1 + 1; j2 < n && h.less(j2, j1) {
			j = j2
		}
		if !h.less(j, i) {
			break
		}
		h.swap(i, j)
		i = j
	}
}

func (h *minHeap) heapify() {
	for i := len(h.a)/2 - 1; i >= 0; i-- {
		h.down(i, len(h.a))
	}
}

type stats struct{ cmps, replaced int }

// topK returns the k largest values of xs (all of them if k > len) in heap order.
// naive == false replaces the root in place and sifts it down; naive == true
// does a Pop followed by a Push. Both replace under exactly the same condition.
func topK(xs []int, k int, naive bool) ([]int, stats) {
	m := min(k, len(xs))
	h := &minHeap{a: slices.Clone(xs[:m])}
	h.heapify() // Floyd, <= 2k comparisons, counted
	replaced := 0
	for _, x := range xs[m:] { // the one pass over the rest of the stream
		h.cmps++ // x against the root: the one comparison every element pays
		if x <= h.a[0] {
			continue
		}
		replaced++
		if naive {
			n := len(h.a) - 1
			h.swap(0, n) // pop: the root goes to the tail ...
			h.down(0, n) // ... the old tail sinks within the first n ...
			h.a = h.a[:n]
			h.a = append(h.a, x) // ... then push x: sift up from the new tail
			h.up(len(h.a) - 1)
		} else {
			h.a[0] = x // overwrite the root and sink it: no pop, no push
			h.down(0, len(h.a))
		}
	}
	return h.a, stats{h.cmps, replaced}
}

// intHeap is the container/heap version: the same replacement through
// heap.Fix(h, 0), with Less counted.
type intHeap struct {
	a    []int
	cmps int
}

func (h *intHeap) Len() int           { return len(h.a) }
func (h *intHeap) Less(i, j int) bool { h.cmps++; return h.a[i] < h.a[j] }
func (h *intHeap) Swap(i, j int)      { h.a[i], h.a[j] = h.a[j], h.a[i] }
func (h *intHeap) Push(x any)         { h.a = append(h.a, x.(int)) }
func (h *intHeap) Pop() any {
	x := h.a[len(h.a)-1]
	h.a = h.a[:len(h.a)-1]
	return x
}

func topKStd(xs []int, k int) stats {
	h := &intHeap{a: slices.Clone(xs[:k])}
	heap.Init(h)
	replaced := 0
	for _, x := range xs[k:] {
		h.cmps++
		if x > h.a[0] {
			replaced++
			h.a[0] = x
			heap.Fix(h, 0)
		}
	}
	return stats{h.cmps, replaced}
}

// shuffled is Fisher-Yates over 0..n-1, hand-written so Rust can repeat it.
func shuffled(r *rand.Rand, n int) []int {
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
	const n = 100000
	r := rand.New(rand.NewPCG(1, 2))
	rnd := shuffled(r, n)
	asc := make([]int, n)
	desc := make([]int, n)
	for i := range asc {
		asc[i], desc[i] = i, n-1-i
	}
	shapes := []struct {
		name string
		xs   []int
	}{{"random", rnd}, {"ascending", asc}, {"descending", desc}}
	ks := []int{10, 100, 1000, 10000}

	fmt.Printf("(a) top-k of n = %d distinct values: comparisons, own min-heap\n", n)
	fmt.Printf("%6s  %-10s %12s %10s %13s %10s %10s\n",
		"k", "shape", "replace-root", "pop+push", "replacements", "k·ln(n/k)", "n·lg k")
	withinBound := true
	var ascRep, ascCmps, ascSlow, rndCmps, rndSlow [4]int
	for ki, k := range ks {
		lg := bits.Len(uint(k)) - 1
		for _, s := range shapes {
			_, fast := topK(s.xs, k, false)
			_, slow := topK(s.xs, k, true)
			if fast.replaced != slow.replaced {
				panic("the two versions must replace under the same condition")
			}
			// Worst case: heapify <= 2k, then n-k roots at 1 comparison, and each
			// replacement sinks through at most floor(lg k) levels at 2 comparisons.
			// pop+push adds the push's sift-up, at most floor(lg k) more.
			boundFast := 2*k + (n-k)*(1+2*lg)
			withinBound = withinBound && fast.cmps <= boundFast && slow.cmps <= boundFast+(n-k)*lg
			fmt.Printf("%6d  %-10s %12d %10d %13d %10.0f %10.0f\n", k, s.name,
				fast.cmps, slow.cmps, fast.replaced,
				float64(k)*math.Log(float64(n)/float64(k)), float64(n)*math.Log2(float64(k)))
			switch s.name {
			case "ascending":
				ascRep[ki], ascCmps[ki], ascSlow[ki] = fast.replaced, fast.cmps, slow.cmps
			case "random":
				rndCmps[ki], rndSlow[ki] = fast.cmps, slow.cmps
			}
		}
	}
	lg := bits.Len(uint(ks[2])) - 1
	fmt.Printf("every run is within (n-k)(1 + 2⌊lg k⌋) + 2k comparisons (pop+push: + ⌊lg k⌋ per element): %v\n", withinBound)
	fmt.Printf("ascending, k = %d: %d of the %d elements after the first k replace the root (%.0f%%),\n",
		ks[2], ascRep[2], n-ks[2], 100*float64(ascRep[2])/float64(n-ks[2]))
	fmt.Printf("  %.2f comparisons per element against 1 + 2⌊lg k⌋ = %d at most: %.2fx n·lg k,\n",
		float64(ascCmps[2])/float64(n), 1+2*lg, float64(ascCmps[2])/(float64(n)*math.Log2(float64(ks[2]))))
	fmt.Printf("  because Θ(n log k) hides the two comparisons per level; random order needs only %.2fx n·lg k\n",
		float64(rndCmps[2])/(float64(n)*math.Log2(float64(ks[2]))))
	fmt.Printf("pop+push costs %.1f%% more than replace-root on random input and %.1f%% more on ascending (k = %d)\n",
		100*(float64(rndSlow[2])/float64(rndCmps[2])-1), 100*(float64(ascSlow[2])/float64(ascCmps[2])-1), ks[2])

	// One permutation is one sample. The count of replacements is a sum of
	// independent coin flips: element i (1-based) beats the root with
	// probability k/i if the order is random, so the mean is k(H_n - H_k).
	fmt.Println()
	const trials = 100
	fmt.Printf("(b) replacements over %d random permutations of n = %d: mean against k·ln(n/k)\n", trials, n)
	fmt.Printf("%6s %9s %12s %10s %12s\n", "k", "mean", "min..max", "k·ln(n/k)", "k(H_n-H_k)")
	for _, k := range ks {
		sum, lo, hi := 0, n, 0
		for range trials {
			_, st := topK(shuffled(r, n), k, false)
			sum += st.replaced
			lo, hi = min(lo, st.replaced), max(hi, st.replaced)
		}
		exact := 0.0
		for i := k + 1; i <= n; i++ {
			exact += float64(k) / float64(i)
		}
		fmt.Printf("%6d %9.1f %12s %10.1f %12.1f\n", k, float64(sum)/trials,
			fmt.Sprintf("%d..%d", lo, hi), float64(k)*math.Log(float64(n)/float64(k)), exact)
	}

	// std lines: the library's own code. container/heap runs the same sift as
	// the loops above, so it must agree exactly; slices.SortFunc is another
	// algorithm entirely, so its count is its own.
	fmt.Println()
	fmt.Println("(c) the same job through the standard library (random permutation, k = 1000)")
	_, own := topK(rnd, 1000, false)
	std := topKStd(rnd, 1000)
	fmt.Printf("own replace-root       %9d comparisons, %5d replacements\n", own.cmps, own.replaced)
	fmt.Printf("container/heap + Fix   %9d comparisons, %5d replacements  (equal to own: %v)\n",
		std.cmps, std.replaced, std == own)
	sortCmps := 0
	sorted := slices.Clone(rnd)
	slices.SortFunc(sorted, func(a, b int) int { sortCmps++; return cmp.Compare(a, b) })
	fmt.Printf("slices.SortFunc, take the first k  %d comparisons (n·lg n = %.0f), and all %d values in memory\n",
		sortCmps, float64(n)*math.Log2(n), n)
	fmt.Printf("the heap used %.1f%% of the sort's comparisons and held 1000 values, not %d\n",
		100*float64(own.cmps)/float64(sortCmps), n)

	// Oracle: sort descending and take the first k. Values come from IntN(15),
	// so there are many ties, and k may exceed n (then the answer is everything).
	fmt.Println()
	mismatches := 0
	const cases = 5000
	for range cases {
		m := 1 + r.IntN(60)
		k := 1 + r.IntN(70)
		xs := make([]int, m)
		for i := range xs {
			xs[i] = r.IntN(15)
		}
		want := slices.Clone(xs)
		slices.SortFunc(want, func(a, b int) int { return cmp.Compare(b, a) })
		want = want[:min(k, m)]
		for _, naive := range []bool{false, true} {
			got, _ := topK(xs, k, naive)
			slices.SortFunc(got, func(a, b int) int { return cmp.Compare(b, a) })
			if !slices.Equal(got, want) {
				mismatches++
			}
		}
	}
	fmt.Printf("(d) oracle, %d random cases (n in 1..60, k in 1..70, values IntN(15)), both versions:\n", cases)
	fmt.Printf("mismatches: %d\n", mismatches)
}
```

**Output:**

```
(a) top-k of n = 100000 distinct values: comparisons, own min-heap
     k  shape      replace-root   pop+push  replacements  k·ln(n/k)     n·lg k
    10  random           100489     100646           105         92     332193
    10  ascending        606615     693689         99990         92     332193
    10  descending       100004     100004             0         92     332193
   100  random           107704     109246           721        691     664386
   100  ascending       1246009    1344397         99900        691     664386
   100  descending       100090     100090             0        691     664386
  1000  random           180945     189504          4721       4605     996578
  1000  ascending       1877504    1976383         99000       4605     996578
  1000  descending       100982     100982             0       4605     996578
 10000  random           658425     692610         23153      23026    1328771
 10000  ascending       2325132    2415111         90000      23026    1328771
 10000  descending       109982     109982             0      23026    1328771
every run is within (n-k)(1 + 2⌊lg k⌋) + 2k comparisons (pop+push: + ⌊lg k⌋ per element): true
ascending, k = 1000: 99000 of the 99000 elements after the first k replace the root (100%),
  18.78 comparisons per element against 1 + 2⌊lg k⌋ = 19 at most: 1.88x n·lg k,
  because Θ(n log k) hides the two comparisons per level; random order needs only 0.18x n·lg k
pop+push costs 4.7% more than replace-root on random input and 5.3% more on ascending (k = 1000)

(b) replacements over 100 random permutations of n = 100000: mean against k·ln(n/k)
     k      mean     min..max  k·ln(n/k)   k(H_n-H_k)
    10      91.5      76..117       92.1         91.6
   100     686.6     628..763      690.8        690.3
  1000    4599.8   4408..4764     4605.2       4604.7
 10000   23040.8 22718..23333    23025.9      23025.4

(c) the same job through the standard library (random permutation, k = 1000)
own replace-root          180945 comparisons,  4721 replacements
container/heap + Fix      180945 comparisons,  4721 replacements  (equal to own: true)
slices.SortFunc, take the first k  1756169 comparisons (n·lg n = 1660964), and all 100000 values in memory
the heap used 10.3% of the sort's comparisons and held 1000 values, not 100000

(d) oracle, 5000 random cases (n in 1..60, k in 1..70, values IntN(15)), both versions:
mismatches: 0
```

**Rust:**

```rust
use std::cell::Cell;
use std::cmp::{Ordering, Reverse};
use std::collections::BinaryHeap;

// Top-K of a stream: keep a MIN-heap of the k largest values seen so far.
// The root is the smallest of the k, i.e. the bar a newcomer must clear. Each
// element costs one comparison with the root; only if it clears the bar does it
// replace the root and sink back into place: Theta(n log k) time, Theta(k) space,
// one pass, and the stream is never stored.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and of its IntN (see lesson
// 01, example 14), so both programs draw the same permutations and the same
// counts come out.
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

// MinHeap is a min-heap of integers whose loops are a verbatim port of Go's
// container/heap `up` and `down` (strict <, left child on ties, a swap at
// every level), counting every call of `less`. Same loops, same counts as Go.
struct MinHeap {
    a: Vec<u64>,
    cmps: u64,
}

impl MinHeap {
    fn less(&mut self, i: usize, j: usize) -> bool {
        self.cmps += 1;
        self.a[i] < self.a[j]
    }

    // Go tests `i == j` because (0-1)/2 is 0 there; a usize has no -1, so the
    // loop condition is `j > 0` instead.
    fn up(&mut self, mut j: usize) {
        while j > 0 {
            let i = (j - 1) / 2;
            if !self.less(j, i) {
                break;
            }
            self.a.swap(i, j);
            j = i;
        }
    }

    fn down(&mut self, mut i: usize, n: usize) {
        loop {
            let j1 = 2 * i + 1;
            if j1 >= n {
                break;
            }
            let mut j = j1;
            let j2 = j1 + 1;
            if j2 < n && self.less(j2, j1) {
                j = j2;
            }
            if !self.less(j, i) {
                break;
            }
            self.a.swap(i, j);
            i = j;
        }
    }

    fn heapify(&mut self) {
        let n = self.a.len();
        for i in (0..n / 2).rev() {
            self.down(i, n);
        }
    }
}

#[derive(PartialEq)]
struct Stats {
    cmps: u64,
    replaced: u64,
}

// top_k returns the k largest values of xs (all of them if k > len) in heap
// order. naive == false replaces the root in place and sifts it down;
// naive == true does a pop followed by a push. Both replace under exactly the
// same condition.
fn top_k(xs: &[u64], k: usize, naive: bool) -> (Vec<u64>, Stats) {
    let m = k.min(xs.len());
    let mut h = MinHeap {
        a: xs[..m].to_vec(),
        cmps: 0,
    };
    h.heapify(); // Floyd, <= 2k comparisons, counted
    let mut replaced = 0;
    for &x in &xs[m..] {
        // the one pass over the rest of the stream
        h.cmps += 1; // x against the root: the one comparison every element pays
        if x <= h.a[0] {
            continue;
        }
        replaced += 1;
        if naive {
            let n = h.a.len() - 1;
            h.a.swap(0, n); // pop: the root goes to the tail ...
            h.down(0, n); // ... the old tail sinks within the first n ...
            h.a.truncate(n);
            h.a.push(x); // ... then push x: sift up from the new tail
            h.up(n);
        } else {
            h.a[0] = x; // overwrite the root and sink it: no pop, no push
            let n = h.a.len();
            h.down(0, n);
        }
    }
    let stats = Stats {
        cmps: h.cmps,
        replaced,
    };
    (h.a, stats)
}

thread_local!(static COMPARES: Cell<u64> = const { Cell::new(0) });

// Counted is the element type for the standard library's heap and sort: its
// `Ord::cmp` bumps a counter, and every comparison operator std uses
// (`<=`, `>=`, `>`) goes through `partial_cmp`, which calls `cmp` once.
#[derive(PartialEq, Eq)]
struct Counted(u64);

impl Ord for Counted {
    fn cmp(&self, other: &Self) -> Ordering {
        COMPARES.with(|c| c.set(c.get() + 1));
        self.0.cmp(&other.0)
    }
}

impl PartialOrd for Counted {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

// BinaryHeap is a max-heap, so Reverse makes its root the smallest value.
// peek_mut hands out a guard: reading through it is free, and the sift-down
// runs when the guard is dropped after a write. A pop-then-push version is
// there to compare against.
fn top_k_std(xs: &[u64], k: usize, peek: bool) -> Stats {
    COMPARES.with(|c| c.set(0));
    let start: Vec<Reverse<Counted>> = xs[..k].iter().map(|&x| Reverse(Counted(x))).collect();
    let mut h = BinaryHeap::from(start); // heapify, in place
    let mut replaced = 0;
    for &x in &xs[k..] {
        if peek {
            let mut top = h.peek_mut().expect("k >= 1");
            if Counted(x) > top.0 {
                // `top.0` reads through Deref: no re-sift if we stop here
                *top = Reverse(Counted(x)); // DerefMut; the sift runs on drop
                replaced += 1;
            }
        } else if Counted(x) > h.peek().expect("k >= 1").0 {
            h.pop();
            h.push(Reverse(Counted(x)));
            replaced += 1;
        }
    }
    Stats {
        cmps: COMPARES.with(Cell::get),
        replaced,
    }
}

// shuffled is Fisher-Yates over 0..n-1, hand-written so Go can repeat it.
fn shuffled(r: &mut Pcg, n: usize) -> Vec<u64> {
    let mut p: Vec<u64> = (0..n as u64).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i as u64 + 1) as usize;
        p.swap(i, j);
    }
    p
}

fn main() {
    const N: usize = 100_000;
    let mut r = Pcg::new(1, 2);
    let rnd = shuffled(&mut r, N);
    let asc: Vec<u64> = (0..N as u64).collect();
    let desc: Vec<u64> = (0..N as u64).rev().collect();
    let shapes = [("random", &rnd), ("ascending", &asc), ("descending", &desc)];
    let ks = [10usize, 100, 1000, 10000];
    let nf = N as f64;

    println!("(a) top-k of n = {N} distinct values: comparisons, own min-heap");
    println!(
        "{:>6}  {:<10} {:>12} {:>10} {:>13} {:>10} {:>10}",
        "k", "shape", "replace-root", "pop+push", "replacements", "k·ln(n/k)", "n·lg k"
    );
    let mut within_bound = true;
    let (mut asc_rep, mut asc_cmps, mut asc_slow) = ([0; 4], [0; 4], [0; 4]);
    let (mut rnd_cmps, mut rnd_slow) = ([0; 4], [0; 4]);
    for (ki, &k) in ks.iter().enumerate() {
        let lg = k.ilog2() as u64;
        for (name, xs) in &shapes {
            let (_, fast) = top_k(xs, k, false);
            let (_, slow) = top_k(xs, k, true);
            assert_eq!(
                fast.replaced, slow.replaced,
                "the two versions must replace under the same condition"
            );
            // Worst case: heapify <= 2k, then n-k roots at 1 comparison, and each
            // replacement sinks through at most floor(lg k) levels at 2 comparisons.
            // pop+push adds the push's sift-up, at most floor(lg k) more.
            let bound_fast = 2 * k as u64 + (N - k) as u64 * (1 + 2 * lg);
            within_bound = within_bound
                && fast.cmps <= bound_fast
                && slow.cmps <= bound_fast + (N - k) as u64 * lg;
            println!(
                "{:>6}  {:<10} {:>12} {:>10} {:>13} {:>10.0} {:>10.0}",
                k,
                name,
                fast.cmps,
                slow.cmps,
                fast.replaced,
                k as f64 * (nf / k as f64).ln(),
                nf * (k as f64).log2()
            );
            match *name {
                "ascending" => {
                    asc_rep[ki] = fast.replaced;
                    asc_cmps[ki] = fast.cmps;
                    asc_slow[ki] = slow.cmps;
                }
                "random" => {
                    rnd_cmps[ki] = fast.cmps;
                    rnd_slow[ki] = slow.cmps;
                }
                _ => {}
            }
        }
    }
    let lg = ks[2].ilog2() as u64;
    println!(
        "every run is within (n-k)(1 + 2⌊lg k⌋) + 2k comparisons (pop+push: + ⌊lg k⌋ per element): {within_bound}"
    );
    println!(
        "ascending, k = {}: {} of the {} elements after the first k replace the root ({:.0}%),",
        ks[2],
        asc_rep[2],
        N - ks[2],
        100.0 * asc_rep[2] as f64 / (N - ks[2]) as f64
    );
    let n_lg_k = nf * (ks[2] as f64).log2();
    println!(
        "  {:.2} comparisons per element against 1 + 2⌊lg k⌋ = {} at most: {:.2}x n·lg k,",
        asc_cmps[2] as f64 / nf,
        1 + 2 * lg,
        asc_cmps[2] as f64 / n_lg_k
    );
    println!(
        "  because Θ(n log k) hides the two comparisons per level; random order needs only {:.2}x n·lg k",
        rnd_cmps[2] as f64 / n_lg_k
    );
    println!(
        "pop+push costs {:.1}% more than replace-root on random input and {:.1}% more on ascending (k = {})",
        100.0 * (rnd_slow[2] as f64 / rnd_cmps[2] as f64 - 1.0),
        100.0 * (asc_slow[2] as f64 / asc_cmps[2] as f64 - 1.0),
        ks[2]
    );

    // One permutation is one sample. The count of replacements is a sum of
    // independent coin flips: element i (1-based) beats the root with
    // probability k/i if the order is random, so the mean is k(H_n - H_k).
    println!();
    const TRIALS: u64 = 100;
    println!(
        "(b) replacements over {TRIALS} random permutations of n = {N}: mean against k·ln(n/k)"
    );
    println!(
        "{:>6} {:>9} {:>12} {:>10} {:>12}",
        "k", "mean", "min..max", "k·ln(n/k)", "k(H_n-H_k)"
    );
    for &k in &ks {
        let (mut sum, mut lo, mut hi) = (0, N as u64, 0);
        for _ in 0..TRIALS {
            let (_, st) = top_k(&shuffled(&mut r, N), k, false);
            sum += st.replaced;
            lo = lo.min(st.replaced);
            hi = hi.max(st.replaced);
        }
        let exact: f64 = (k + 1..=N).map(|i| k as f64 / i as f64).sum();
        println!(
            "{:>6} {:>9.1} {:>12} {:>10.1} {:>12.1}",
            k,
            sum as f64 / TRIALS as f64,
            format!("{lo}..{hi}"),
            k as f64 * (nf / k as f64).ln(),
            exact
        );
    }

    // std lines: the library's own code. Go's container/heap runs the same
    // sift as the loops above; Rust's BinaryHeap has its own sifts, so this
    // block says what they cost instead of assuming.
    println!();
    println!("(c) the same job through the standard library (random permutation, k = 1000)");
    let (_, own) = top_k(&rnd, 1000, false);
    let (_, own_slow) = top_k(&rnd, 1000, true);
    let std_peek = top_k_std(&rnd, 1000, true);
    let std_pop = top_k_std(&rnd, 1000, false);
    println!(
        "own replace-root       {:>9} comparisons, {:>5} replacements",
        own.cmps, own.replaced
    );
    println!(
        "BinaryHeap + peek_mut  {:>9} comparisons, {:>5} replacements  (equal to own: {})",
        std_peek.cmps,
        std_peek.replaced,
        std_peek == own
    );
    println!(
        "own pop+push           {:>9} comparisons, {:>5} replacements",
        own_slow.cmps, own_slow.replaced
    );
    println!(
        "BinaryHeap pop + push  {:>9} comparisons, {:>5} replacements  (equal to own: {})",
        std_pop.cmps,
        std_pop.replaced,
        std_pop == own_slow
    );
    let mut sort_cmps = 0u64;
    let mut sorted = rnd.clone();
    sorted.sort_unstable_by(|a, b| {
        sort_cmps += 1;
        a.cmp(b)
    });
    println!(
        "sort_unstable_by, take the first k  {} comparisons (n·lg n = {:.0}), and all {} values in memory",
        sort_cmps,
        nf * nf.log2(),
        N
    );
    println!(
        "the heap used {:.1}% of the sort's comparisons and held 1000 values, not {}",
        100.0 * own.cmps as f64 / sort_cmps as f64,
        N
    );

    // Oracle: sort descending and take the first k. Values come from IntN(15),
    // so there are many ties, and k may exceed n (then the answer is everything).
    println!();
    let mut mismatches = 0;
    const CASES: u32 = 5000;
    for _ in 0..CASES {
        let m = 1 + r.int_n(60) as usize;
        let k = 1 + r.int_n(70) as usize;
        let xs: Vec<u64> = (0..m).map(|_| r.int_n(15)).collect();
        let mut want = xs.clone();
        want.sort_unstable_by(|a, b| b.cmp(a));
        want.truncate(k.min(m));
        for naive in [false, true] {
            let (mut got, _) = top_k(&xs, k, naive);
            got.sort_unstable_by(|a, b| b.cmp(a));
            if got != want {
                mismatches += 1;
            }
        }
    }
    println!(
        "(d) oracle, {CASES} random cases (n in 1..60, k in 1..70, values IntN(15)), both versions:"
    );
    println!("mismatches: {mismatches}");
}
```

**Rust output:**

```
(a) top-k of n = 100000 distinct values: comparisons, own min-heap
     k  shape      replace-root   pop+push  replacements  k·ln(n/k)     n·lg k
    10  random           100489     100646           105         92     332193
    10  ascending        606615     693689         99990         92     332193
    10  descending       100004     100004             0         92     332193
   100  random           107704     109246           721        691     664386
   100  ascending       1246009    1344397         99900        691     664386
   100  descending       100090     100090             0        691     664386
  1000  random           180945     189504          4721       4605     996578
  1000  ascending       1877504    1976383         99000       4605     996578
  1000  descending       100982     100982             0       4605     996578
 10000  random           658425     692610         23153      23026    1328771
 10000  ascending       2325132    2415111         90000      23026    1328771
 10000  descending       109982     109982             0      23026    1328771
every run is within (n-k)(1 + 2⌊lg k⌋) + 2k comparisons (pop+push: + ⌊lg k⌋ per element): true
ascending, k = 1000: 99000 of the 99000 elements after the first k replace the root (100%),
  18.78 comparisons per element against 1 + 2⌊lg k⌋ = 19 at most: 1.88x n·lg k,
  because Θ(n log k) hides the two comparisons per level; random order needs only 0.18x n·lg k
pop+push costs 4.7% more than replace-root on random input and 5.3% more on ascending (k = 1000)

(b) replacements over 100 random permutations of n = 100000: mean against k·ln(n/k)
     k      mean     min..max  k·ln(n/k)   k(H_n-H_k)
    10      91.5      76..117       92.1         91.6
   100     686.6     628..763      690.8        690.3
  1000    4599.8   4408..4764     4605.2       4604.7
 10000   23040.8 22718..23333    23025.9      23025.4

(c) the same job through the standard library (random permutation, k = 1000)
own replace-root          180945 comparisons,  4721 replacements
BinaryHeap + peek_mut     180945 comparisons,  4721 replacements  (equal to own: true)
own pop+push              189504 comparisons,  4721 replacements
BinaryHeap pop + push     157494 comparisons,  4721 replacements  (equal to own: false)
sort_unstable_by, take the first k  1721445 comparisons (n·lg n = 1660964), and all 100000 values in memory
the heap used 10.5% of the sort's comparisons and held 1000 values, not 100000

(d) oracle, 5000 random cases (n in 1..60, k in 1..70, values IntN(15)), both versions:
mismatches: 0
```

> **Go vs Rust:** Parts (a), (b) and (d) are identical to the Go run (diffed): the own heap is the same `up`/`down` loops, and Rust's `usize` has no -1, so `up` loops on `j > 0` instead of Go's `i == j` test. In (c), `BinaryHeap` + `peek_mut` makes exactly the own heap's 180,945 comparisons, the same as Go's `container/heap` + `Fix`, because the guard's `Drop` runs the textbook two-comparisons-per-level sift-down, while std's `pop` sifts bottom-up with one comparison per level, so pop+push costs 157,494 against the own heap's 189,504 (both read in the source, not run). `sort_unstable_by` took 1,721,445 comparisons against `slices.SortFunc`'s 1,756,169 (different algorithms), and counting through std needs a `Counted` newtype whose `Ord::cmp` bumps a counter, where Go counts inside `Less` or the comparator.

**Complexity:** Θ(n log k) time, Θ(k) space, one pass; the root test is one comparison per element, and a replacement costs at most 2⌊lg k⌋. On random order the replacements averaged within 1% of k ln(n/k) over 100 permutations (about 4,600 at k = 1000), so the count stays near 0.18 n lg k; ascending order replaces every element and reaches 1.88 n lg k, about 19 comparisons per element. `container/heap` with `Fix(h, 0)` makes exactly the own heap's comparisons, and sorting everything costs about 9.7 times as many for k = 1000.

---

## 8. Merging k sorted lists

`🟡 medium` · *k-way merge / laziness*

A heap of the k list heads merges N elements in Θ(N log k): pop the smallest, replace it with the next element of the same list, sift down. It is **not** the cheapest in comparisons, because a balanced tree of pairwise merges does at most N⌈lg k⌉; what the heap has is **laziness**, since it can hand out the first m values after about k + m log k work while the eager strategies have produced nothing until they finish. And when the lists have unequal sizes, the best pairwise order is Huffman's: always merge the two shortest, which is itself a heap problem. ([Lesson 13](../../13-efficient-sorts.md), example 14 used this merge for runs on disk; this is the in-memory comparison of strategies.)

**Steps:**

1. Write four strategies that each return the merged list and count comparisons between elements: the heap of heads, a sequential fold, balanced pairwise rounds, and smallest-first (an own heap of `(length, id)`).
2. Run them on 100,000 values over k = 2…1024 random lists, then on k = 32 lists with even, geometric and one-big sizes.
3. Read the counts against N lg k, then take only the first m outputs of the heap merge and check every strategy against concatenate-and-sort.

```go
package main

import (
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
)

// Merging k sorted lists four ways. The heap of heads is the lazy one; pairwise
// merging is eager and, for even sizes, cheaper in comparisons; and when the
// lists are unequal, the cheapest pairwise order is Huffman's: always merge the
// two shortest. Every strategy returns the merged list and counts the
// comparisons between list elements.

// heapOf is a generic binary min-heap whose up/down loops are copied from
// container/heap (strict less, left child on ties, a swap at every level) and
// which counts every call of lt. The Rust twin ports the same loops.
type heapOf[T any] struct {
	a    []T
	lt   func(x, y T) bool
	cmps int
}

func (h *heapOf[T]) less(i, j int) bool { h.cmps++; return h.lt(h.a[i], h.a[j]) }
func (h *heapOf[T]) swap(i, j int)      { h.a[i], h.a[j] = h.a[j], h.a[i] }

func (h *heapOf[T]) up(j int) {
	for {
		i := (j - 1) / 2
		if i == j || !h.less(j, i) {
			break
		}
		h.swap(i, j)
		j = i
	}
}

func (h *heapOf[T]) down(i, n int) {
	for {
		j1 := 2*i + 1
		if j1 >= n {
			break
		}
		j := j1
		if j2 := j1 + 1; j2 < n && h.less(j2, j1) {
			j = j2
		}
		if !h.less(j, i) {
			break
		}
		h.swap(i, j)
		i = j
	}
}

func (h *heapOf[T]) heapify() {
	for i := len(h.a)/2 - 1; i >= 0; i-- {
		h.down(i, len(h.a))
	}
}

func (h *heapOf[T]) push(x T) { h.a = append(h.a, x); h.up(len(h.a) - 1) }

func (h *heapOf[T]) pop() T {
	n := len(h.a) - 1
	h.swap(0, n)
	h.down(0, n)
	x := h.a[n]
	h.a = h.a[:n]
	return x
}

// head is the front of one list. The key (val, list) is a strict total order:
// no two heads share a list, so equal values still compare, and the merge is
// stable (on a tie the lower-numbered list goes first).
type head struct{ val, list, pos int }

func headLess(x, y head) bool { return x.val < y.val || x.val == y.val && x.list < y.list }

// kmerge is a LAZY k-way merge: it holds one head per non-empty list and does
// work only when asked for the next value.
type kmerge struct {
	lists [][]int
	h     *heapOf[head]
}

func newKMerge(lists [][]int) *kmerge {
	h := &heapOf[head]{lt: headLess}
	for i, l := range lists {
		if len(l) > 0 {
			h.a = append(h.a, head{l[0], i, 0})
		}
	}
	h.heapify() // Floyd: at most 2k comparisons, paid up front
	return &kmerge{lists, h}
}

func (m *kmerge) next() (int, bool) {
	h := m.h
	if len(h.a) == 0 {
		return 0, false
	}
	top := h.a[0]
	if l := m.lists[top.list]; top.pos+1 < len(l) {
		h.a[0] = head{l[top.pos+1], top.list, top.pos + 1} // replace the root ...
		h.down(0, len(h.a))                                // ... and sink it
	} else {
		n := len(h.a) - 1 // list exhausted: the root leaves the heap
		h.swap(0, n)
		h.a = h.a[:n]
		h.down(0, n)
	}
	return top.val, true
}

type result struct {
	out          []int
	cmps, writes int // element comparisons; elements written by merge steps
}

func heapMerge(lists [][]int) result {
	m := newKMerge(lists)
	var out []int
	for v, ok := m.next(); ok; v, ok = m.next() {
		out = append(out, v)
	}
	return result{out, m.h.cmps, len(out)}
}

// merge2 merges two sorted lists: one comparison per element it emits while
// both lists are non-empty, then the rest of the longer list is copied free.
func merge2(a, b []int, cmps *int) []int {
	out := make([]int, 0, len(a)+len(b))
	i, j := 0, 0
	for i < len(a) && j < len(b) {
		*cmps++
		if b[j] < a[i] {
			out = append(out, b[j])
			j++
		} else {
			out = append(out, a[i])
			i++
		}
	}
	out = append(out, a[i:]...)
	return append(out, b[j:]...)
}

func sequential(lists [][]int) result {
	var res result
	acc := lists[0]
	for _, l := range lists[1:] {
		acc = merge2(acc, l, &res.cmps)
		res.writes += len(acc)
	}
	res.out = acc
	return res
}

func balanced(lists [][]int) result {
	var res result
	cur := slices.Clone(lists)
	for len(cur) > 1 { // one round merges neighbouring pairs
		var next [][]int
		for i := 0; i+1 < len(cur); i += 2 {
			m := merge2(cur[i], cur[i+1], &res.cmps)
			res.writes += len(m)
			next = append(next, m)
		}
		if len(cur)%2 == 1 {
			next = append(next, cur[len(cur)-1])
		}
		cur = next
	}
	res.out = cur[0]
	return res
}

type sized struct{ n, id int }

// smallestFirst is Huffman's order: a heap of (length, id) pairs, pop the two
// shortest, merge them, push the result back.
func smallestFirst(lists [][]int) result {
	var res result
	all := slices.Clone(lists) // ids index into all; merged lists are appended
	h := &heapOf[sized]{lt: func(x, y sized) bool { return x.n < y.n || x.n == y.n && x.id < y.id }}
	for id, l := range all {
		h.a = append(h.a, sized{len(l), id})
	}
	h.heapify()
	for len(h.a) > 1 {
		x, y := h.pop(), h.pop()
		m := merge2(all[x.id], all[y.id], &res.cmps)
		res.writes += len(m)
		all[x.id], all[y.id] = nil, nil
		all = append(all, m)
		h.push(sized{len(m), len(all) - 1})
	}
	res.out = all[h.a[0].id]
	return res
}

var strategies = []struct {
	name string
	run  func([][]int) result
}{{"heap", heapMerge}, {"sequential", sequential}, {"balanced", balanced}, {"smallest-first", smallestFirst}}

// deal makes lists of the given sizes with random sorted values.
func deal(r *rand.Rand, sizes []int, maxVal int) [][]int {
	lists := make([][]int, len(sizes))
	for i, n := range sizes {
		lists[i] = make([]int, n)
		for j := range lists[i] {
			lists[i][j] = r.IntN(maxVal)
		}
		slices.Sort(lists[i])
	}
	return lists
}

// runAll runs every strategy, counts any output that differs from
// concat+sort, and returns the four results.
func runAll(lists [][]int, mismatches *int) []result {
	want := slices.Sorted(slices.Values(slices.Concat(lists...)))
	res := make([]result, len(strategies))
	for i, s := range strategies {
		res[i] = s.run(lists)
		if !slices.Equal(res[i].out, want) {
			*mismatches++
		}
	}
	return res
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	const N = 100000
	mismatches := 0

	fmt.Printf("(a) N = %d values IntN(1000000) dealt to k random lists: comparisons between elements\n", N)
	fmt.Printf("%5s %10s %11s %9s %15s %9s %9s\n", "k", "heap", "sequential", "balanced", "smallest-first", "N·lg k", "N⌈lg k⌉")
	balancedOK, balancedLeHeap, foldWinsUpTo := true, true, 0
	var lazyLists [][]int
	var res []result
	for _, k := range []int{2, 4, 16, 64, 256, 1024} {
		lists := make([][]int, k)
		for i := 0; i < N; i++ {
			v := r.IntN(1000000)
			l := r.IntN(k)
			lists[l] = append(lists[l], v)
		}
		for _, l := range lists {
			slices.Sort(l)
		}
		res = runAll(lists, &mismatches)
		ceil := bits.Len(uint(k - 1)) // ⌈lg k⌉
		balancedOK = balancedOK && res[2].cmps <= N*ceil
		balancedLeHeap = balancedLeHeap && res[2].cmps <= res[0].cmps
		if res[1].cmps <= res[0].cmps {
			foldWinsUpTo = k // the fold is no dearer than the heap at this k
		}
		fmt.Printf("%5d %10d %11d %9d %15d %9.0f %9d\n", k, res[0].cmps, res[1].cmps, res[2].cmps,
			res[3].cmps, N*math.Log2(float64(k)), N*ceil)
		lazyLists = lists
	}
	fmt.Printf("balanced <= N⌈lg k⌉ in every row: %v;  balanced <= heap in every row: %v\n", balancedOK, balancedLeHeap)
	fmt.Printf("the fold costs no more than the heap up to k = %d; at k = 1024 it needs %.0fx the comparisons\n",
		foldWinsUpTo, float64(res[1].cmps)/float64(res[0].cmps))
	fmt.Printf("of the heap (Θ(Nk) against Θ(N log k)), and the heap needs %.2fx those of the balanced tree\n",
		float64(res[0].cmps)/float64(res[2].cmps))

	// Same N, k = 32, three size shapes. writes = elements written by pairwise
	// merge steps (the sum of every merge's output length): the Huffman order
	// minimises it, and comparisons follow it closely.
	fmt.Println()
	fmt.Println("(b) k = 32 lists, three size shapes (comparisons; then elements written by the merge steps)")
	even, geo, big := make([]int, 32), make([]int, 32), make([]int, 32)
	for i := range even {
		even[i], geo[i], big[i] = N/32, max(N>>(i+1), 1), 100
	}
	big[0] = N - 31*100
	fmt.Printf("%-10s %7s %8s %11s %9s %15s | %11s %9s %15s\n", "shape", "N", "heap", "sequential",
		"balanced", "smallest-first", "sequential", "balanced", "smallest-first")
	fewest, skewHeapWins := true, true
	for _, sh := range []struct {
		name  string
		sizes []int
	}{{"even", even}, {"geometric", geo}, {"one-big", big}} {
		res = runAll(deal(r, sh.sizes, 1000000), &mismatches)
		fewest = fewest && res[3].writes <= res[1].writes && res[3].writes <= res[2].writes
		if sh.name != "even" {
			skewHeapWins = skewHeapWins && res[0].cmps < res[2].cmps
		}
		fmt.Printf("%-10s %7d %8d %11d %9d %15d | %11d %9d %15d\n", sh.name, len(res[0].out), res[0].cmps,
			res[1].cmps, res[2].cmps, res[3].cmps, res[1].writes, res[2].writes, res[3].writes)
	}
	fmt.Printf("smallest-first writes the fewest elements of the three pairwise orders in every shape: %v\n", fewest)
	fmt.Printf("on the two skewed shapes the heap is below balanced pairing, which ignores sizes: %v\n", skewHeapWins)
	fmt.Printf("one-big: balanced %d comparisons, smallest-first %d (%.1fx fewer), the heap %d\n",
		res[2].cmps, res[3].cmps, float64(res[2].cmps)/float64(res[3].cmps), res[0].cmps)

	// Laziness: heapify costs at most 2k comparisons and each output at most
	// 2⌊lg k⌋, so the first m values arrive after at most 2k + 2m⌊lg k⌋. The
	// eager strategies have produced nothing until they are done.
	fmt.Println()
	k := len(lazyLists)
	lg := bits.Len(uint(k)) - 1
	eager := runAll(lazyLists, &mismatches)
	want := slices.Sorted(slices.Values(slices.Concat(lazyLists...)))
	fmt.Printf("(c) the first m outputs of the k = %d heap merge (cumulative comparisons, heapify included)\n", k)
	fmt.Printf("%9s %12s %16s %14s\n", "m", "comparisons", "bound 2k+2m⌊lg k⌋", "per output")
	m, got := newKMerge(lazyLists), 0
	prefixOK, boundOK := true, true
	fmt.Printf("%9d %12d %16d %14s\n", 0, m.h.cmps, 2*k, "-")
	for _, target := range []int{1, 10, 100, 1000, 10000, N} {
		for ; got < target; got++ {
			v, _ := m.next()
			prefixOK = prefixOK && v == want[got]
		}
		bound := 2*k + 2*target*lg
		boundOK = boundOK && m.h.cmps <= bound
		fmt.Printf("%9d %12d %16d %14.2f\n", target, m.h.cmps, bound, float64(m.h.cmps)/float64(target))
	}
	fmt.Printf("every prefix equals the sorted order: %v;  every row within the bound: %v\n", prefixOK, boundOK)
	first := newKMerge(lazyLists)
	first.next()
	fmt.Printf("the first output cost %d comparisons; balanced pairwise needs %d and the fold %d before any output exists\n",
		first.h.cmps, eager[2].cmps, eager[1].cmps)

	// Oracle: 2,000 tiny configurations, empty lists included, values IntN(10)
	// so there are many ties. Besides equality with concat+sort, Huffman's
	// order must write no more elements than either other pairwise order.
	fmt.Println()
	const cases = 2000
	huffmanOK := true
	for range cases {
		sizes := make([]int, 1+r.IntN(8))
		for i := range sizes {
			sizes[i] = r.IntN(7)
		}
		res = runAll(deal(r, sizes, 10), &mismatches)
		huffmanOK = huffmanOK && res[3].writes <= res[1].writes && res[3].writes <= res[2].writes
	}
	fmt.Printf("(d) oracle: all four strategies, %d tiny cases (k in 1..8, sizes 0..6, values IntN(10)) and the runs above\n", cases)
	fmt.Printf("mismatches: %d;  smallest-first never writes more than balanced or the fold: %v\n", mismatches, huffmanOK)
}
```

**Output:**

```
(a) N = 100000 values IntN(1000000) dealt to k random lists: comparisons between elements
    k       heap  sequential  balanced  smallest-first    N·lg k   N⌈lg k⌉
    2      99999       99999     99999           99999    100000    100000
    4     249948      225092    199994          199995    200000    200000
   16     562923      839559    399965          399971    400000    400000
   64     921426     3248546    599873          599867    600000    600000
  256    1308188    12813802    799483          799479    800000    800000
 1024    1700121    50759014    997952          998045   1000000   1000000
balanced <= N⌈lg k⌉ in every row: true;  balanced <= heap in every row: true
the fold costs no more than the heap up to k = 4; at k = 1024 it needs 30x the comparisons
of the heap (Θ(Nk) against Θ(N log k)), and the heap needs 1.70x those of the balanced tree

(b) k = 32 lists, three size shapes (comparisons; then elements written by the merge steps)
shape            N     heap  sequential  balanced  smallest-first |  sequential  balanced  smallest-first
even        100000   737441     1646076    499934          499934 |     1646875    500000          500000
geometric   100010   394483     2093319    497765          200174 |     3050012    500050          200219
one-big     100000   223147     3033184    499432          115329 |     3053500    500000          115400
smallest-first writes the fewest elements of the three pairwise orders in every shape: true
on the two skewed shapes the heap is below balanced pairing, which ignores sizes: true
one-big: balanced 499432 comparisons, smallest-first 115329 (4.3x fewer), the heap 223147

(c) the first m outputs of the k = 1024 heap merge (cumulative comparisons, heapify included)
        m  comparisons bound 2k+2m⌊lg k⌋     per output
        0         1920             2048              -
        1         1938             2068        1938.00
       10         2092             2248         209.20
      100         3600             4048          36.00
     1000        18862            22048          18.86
    10000       171838           202048          17.18
   100000      1700121          2002048          17.00
every prefix equals the sorted order: true;  every row within the bound: true
the first output cost 1938 comparisons; balanced pairwise needs 997952 and the fold 50759014 before any output exists

(d) oracle: all four strategies, 2000 tiny cases (k in 1..8, sizes 0..6, values IntN(10)) and the runs above
mismatches: 0;  smallest-first never writes more than balanced or the fold: true
```

**Rust:**

```rust
use std::cell::Cell;
use std::cmp::{Ordering, Reverse};
use std::collections::BinaryHeap;
use std::collections::binary_heap::PeekMut;

// Merging k sorted lists four ways. The heap of heads is the lazy one; pairwise
// merging is eager and, for even sizes, cheaper in comparisons; and when the
// lists are unequal, the cheapest pairwise order is Huffman's: always merge the
// two shortest. Every strategy returns the merged list and counts the
// comparisons between list elements.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and of its IntN (see lesson
// 01, example 14), so both programs deal the same lists and the same counts
// come out.
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

// Heap is a generic binary min-heap whose up/down loops are a verbatim port of
// Go's container/heap (strict less, left child on ties, a swap at every
// level), counting every call of `lt`. Same loops, same counts as Go.
struct Heap<T> {
    a: Vec<T>,
    lt: fn(&T, &T) -> bool,
    cmps: u64,
}

impl<T> Heap<T> {
    fn new(a: Vec<T>, lt: fn(&T, &T) -> bool) -> Self {
        Heap { a, lt, cmps: 0 }
    }

    fn less(&mut self, i: usize, j: usize) -> bool {
        self.cmps += 1;
        (self.lt)(&self.a[i], &self.a[j])
    }

    // A usize has no -1, so the loop condition is `j > 0` where Go tests i == j.
    fn up(&mut self, mut j: usize) {
        while j > 0 {
            let i = (j - 1) / 2;
            if !self.less(j, i) {
                break;
            }
            self.a.swap(i, j);
            j = i;
        }
    }

    fn down(&mut self, mut i: usize, n: usize) {
        loop {
            let j1 = 2 * i + 1;
            if j1 >= n {
                break;
            }
            let mut j = j1;
            let j2 = j1 + 1;
            if j2 < n && self.less(j2, j1) {
                j = j2;
            }
            if !self.less(j, i) {
                break;
            }
            self.a.swap(i, j);
            i = j;
        }
    }

    fn heapify(&mut self) {
        let n = self.a.len();
        for i in (0..n / 2).rev() {
            self.down(i, n);
        }
    }

    fn push(&mut self, x: T) {
        self.a.push(x);
        self.up(self.a.len() - 1);
    }

    fn pop(&mut self) -> T {
        let n = self.a.len() - 1;
        self.a.swap(0, n);
        self.down(0, n);
        self.a.pop().expect("pop on a non-empty heap")
    }
}

// Head is the front of one list. The key (val, list) is a strict total order:
// no two heads share a list, so equal values still compare, and the merge is
// stable (on a tie the lower-numbered list goes first).
#[derive(Clone, Copy)]
struct Head {
    val: i64,
    list: usize,
    pos: usize,
}

fn head_less(x: &Head, y: &Head) -> bool {
    x.val < y.val || x.val == y.val && x.list < y.list
}

// KMerge is a LAZY k-way merge: it holds one head per non-empty list and does
// work only when asked for the next value. In Rust that is just an Iterator,
// so `.take(m)` is the "first m outputs".
struct KMerge<'a> {
    lists: &'a [Vec<i64>],
    h: Heap<Head>,
    removal_cmps: u64, // comparisons spent when a list ran out (only read in (e))
}

impl<'a> KMerge<'a> {
    fn new(lists: &'a [Vec<i64>]) -> Self {
        let heads = lists
            .iter()
            .enumerate()
            .filter(|(_, l)| !l.is_empty())
            .map(|(i, l)| Head {
                val: l[0],
                list: i,
                pos: 0,
            })
            .collect();
        let mut h = Heap::new(heads, head_less);
        h.heapify(); // Floyd: at most 2k comparisons, paid up front
        KMerge {
            lists,
            h,
            removal_cmps: 0,
        }
    }
}

impl Iterator for KMerge<'_> {
    type Item = i64;

    fn next(&mut self) -> Option<i64> {
        let h = &mut self.h;
        let top = *h.a.first()?;
        let l = &self.lists[top.list];
        if top.pos + 1 < l.len() {
            h.a[0] = Head {
                val: l[top.pos + 1],
                list: top.list,
                pos: top.pos + 1,
            }; // replace the root ...
            let n = h.a.len();
            h.down(0, n); // ... and sink it
        } else {
            let before = h.cmps;
            let n = h.a.len() - 1; // list exhausted: the root leaves the heap
            h.a.swap(0, n);
            h.a.truncate(n);
            h.down(0, n);
            self.removal_cmps += h.cmps - before;
        }
        Some(top.val)
    }
}

struct Res {
    out: Vec<i64>,
    cmps: u64,
    writes: u64, // element comparisons; elements written by merge steps
}

fn heap_merge(lists: &[Vec<i64>]) -> Res {
    let mut m = KMerge::new(lists);
    let out: Vec<i64> = m.by_ref().collect();
    let writes = out.len() as u64;
    Res {
        out,
        cmps: m.h.cmps,
        writes,
    }
}

// merge2 merges two sorted lists: one comparison per element it emits while
// both lists are non-empty, then the rest of the longer list is copied free.
fn merge2(a: &[i64], b: &[i64], cmps: &mut u64) -> Vec<i64> {
    let mut out = Vec::with_capacity(a.len() + b.len());
    let (mut i, mut j) = (0, 0);
    while i < a.len() && j < b.len() {
        *cmps += 1;
        if b[j] < a[i] {
            out.push(b[j]);
            j += 1;
        } else {
            out.push(a[i]);
            i += 1;
        }
    }
    out.extend_from_slice(&a[i..]);
    out.extend_from_slice(&b[j..]);
    out
}

fn sequential(lists: &[Vec<i64>]) -> Res {
    let (mut cmps, mut writes) = (0, 0);
    let mut acc = lists[0].clone();
    for l in &lists[1..] {
        acc = merge2(&acc, l, &mut cmps);
        writes += acc.len() as u64;
    }
    Res {
        out: acc,
        cmps,
        writes,
    }
}

fn balanced(lists: &[Vec<i64>]) -> Res {
    let (mut cmps, mut writes) = (0, 0);
    let mut cur = lists.to_vec();
    while cur.len() > 1 {
        // one round merges neighbouring pairs; an odd one out is carried over
        let mut next = Vec::new();
        for pair in cur.chunks(2) {
            if let [a, b] = pair {
                let m = merge2(a, b, &mut cmps);
                writes += m.len() as u64;
                next.push(m);
            } else {
                next.push(pair[0].clone());
            }
        }
        cur = next;
    }
    Res {
        out: cur.swap_remove(0),
        cmps,
        writes,
    }
}

// smallest_first is Huffman's order: a heap of (length, id) pairs, pop the two
// shortest, merge them, push the result back. Tuples compare lexicographically,
// which is exactly the strict (length, id) order.
fn smallest_first(lists: &[Vec<i64>]) -> Res {
    let (mut cmps, mut writes) = (0, 0);
    let mut all = lists.to_vec(); // ids index into all; merged lists are appended
    let sizes = all.iter().enumerate().map(|(id, l)| (l.len(), id));
    let mut h = Heap::new(sizes.collect(), |x: &(usize, usize), y| x < y);
    h.heapify();
    while h.a.len() > 1 {
        let (x, y) = (h.pop(), h.pop());
        let (a, b) = (std::mem::take(&mut all[x.1]), std::mem::take(&mut all[y.1]));
        let m = merge2(&a, &b, &mut cmps);
        writes += m.len() as u64;
        h.push((m.len(), all.len()));
        all.push(m);
    }
    let id = h.a[0].1;
    Res {
        out: std::mem::take(&mut all[id]),
        cmps,
        writes,
    }
}

type Strategy = fn(&[Vec<i64>]) -> Res;
const STRATEGIES: [Strategy; 4] = [heap_merge, sequential, balanced, smallest_first];

// deal makes lists of the given sizes with random sorted values.
fn deal(r: &mut Pcg, sizes: &[usize], max_val: u64) -> Vec<Vec<i64>> {
    sizes
        .iter()
        .map(|&n| {
            let mut l: Vec<i64> = (0..n).map(|_| r.int_n(max_val) as i64).collect();
            l.sort_unstable();
            l
        })
        .collect()
}

// run_all runs every strategy, counts any output that differs from
// concat+sort, and returns the four results.
fn run_all(lists: &[Vec<i64>], mismatches: &mut u32) -> Vec<Res> {
    let mut want: Vec<i64> = lists.concat();
    want.sort_unstable();
    STRATEGIES
        .iter()
        .map(|run| {
            let res = run(lists);
            if res.out != want {
                *mismatches += 1;
            }
            res
        })
        .collect()
}

thread_local!(static COMPARES: Cell<u64> = const { Cell::new(0) });

// StdHead is the head type for the standard library's heap: its `Ord::cmp`
// uses the same (val, list) key and bumps a counter, and every comparison
// operator BinaryHeap uses goes through `partial_cmp`, which calls `cmp` once.
#[derive(PartialEq, Eq)]
struct StdHead {
    val: i64,
    list: usize,
    pos: usize,
}

impl Ord for StdHead {
    fn cmp(&self, other: &Self) -> Ordering {
        COMPARES.with(|c| c.set(c.get() + 1));
        (self.val, self.list).cmp(&(other.val, other.list))
    }
}

impl PartialOrd for StdHead {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

// std_merge: the heap of heads on BinaryHeap<Reverse<StdHead>>, the heads kept
// in slices of the caller's lists (no copying). peek: replace the root through
// the PeekMut guard (the sift runs when it drops); otherwise pop, then push.
fn std_merge(lists: &[Vec<i64>], peek: bool) -> (Vec<i64>, u64, u64) {
    COMPARES.with(|c| c.set(0));
    let heads: Vec<Reverse<StdHead>> = lists
        .iter()
        .enumerate()
        .filter(|(_, l)| !l.is_empty())
        .map(|(list, l)| {
            Reverse(StdHead {
                val: l[0],
                list,
                pos: 0,
            })
        })
        .collect();
    let mut h = BinaryHeap::from(heads);
    let mut out = Vec::new();
    let mut removal_cmps = 0; // comparisons spent in PeekMut::pop, when a list ran out
    if peek {
        while let Some(mut top) = h.peek_mut() {
            let (val, list, pos) = (top.0.val, top.0.list, top.0.pos);
            out.push(val);
            if let Some(&next) = lists[list].get(pos + 1) {
                *top = Reverse(StdHead {
                    val: next,
                    list,
                    pos: pos + 1,
                });
            } else {
                let before = COMPARES.with(Cell::get);
                PeekMut::pop(top);
                removal_cmps += COMPARES.with(Cell::get) - before;
            }
        }
    } else {
        while let Some(Reverse(top)) = h.pop() {
            out.push(top.val);
            if let Some(&next) = lists[top.list].get(top.pos + 1) {
                h.push(Reverse(StdHead {
                    val: next,
                    list: top.list,
                    pos: top.pos + 1,
                }));
            }
        }
    }
    (out, COMPARES.with(Cell::get), removal_cmps)
}

fn main() {
    let mut r = Pcg::new(1, 2);
    const N: usize = 100_000;
    let mut mismatches = 0;

    println!(
        "(a) N = {N} values IntN(1000000) dealt to k random lists: comparisons between elements"
    );
    println!(
        "{:>5} {:>10} {:>11} {:>9} {:>15} {:>9} {:>9}",
        "k", "heap", "sequential", "balanced", "smallest-first", "N·lg k", "N⌈lg k⌉"
    );
    let (mut balanced_ok, mut balanced_le_heap, mut fold_wins_up_to) = (true, true, 0);
    let mut lazy_lists = Vec::new();
    let mut lists64 = Vec::new();
    let mut res = Vec::new();
    for k in [2usize, 4, 16, 64, 256, 1024] {
        let mut lists: Vec<Vec<i64>> = vec![Vec::new(); k];
        for _ in 0..N {
            let v = r.int_n(1_000_000) as i64;
            let l = r.int_n(k as u64) as usize;
            lists[l].push(v);
        }
        for l in &mut lists {
            l.sort_unstable();
        }
        res = run_all(&lists, &mut mismatches);
        let ceil = (usize::BITS - (k - 1).leading_zeros()) as u64; // ⌈lg k⌉
        balanced_ok = balanced_ok && res[2].cmps <= N as u64 * ceil;
        balanced_le_heap = balanced_le_heap && res[2].cmps <= res[0].cmps;
        if res[1].cmps <= res[0].cmps {
            fold_wins_up_to = k; // the fold is no dearer than the heap at this k
        }
        println!(
            "{:>5} {:>10} {:>11} {:>9} {:>15} {:>9.0} {:>9}",
            k,
            res[0].cmps,
            res[1].cmps,
            res[2].cmps,
            res[3].cmps,
            N as f64 * (k as f64).log2(),
            N as u64 * ceil
        );
        if k == 64 {
            lists64 = lists.clone();
        }
        lazy_lists = lists;
    }
    println!(
        "balanced <= N⌈lg k⌉ in every row: {balanced_ok};  balanced <= heap in every row: {balanced_le_heap}"
    );
    println!(
        "the fold costs no more than the heap up to k = {fold_wins_up_to}; at k = 1024 it needs {:.0}x the comparisons",
        res[1].cmps as f64 / res[0].cmps as f64
    );
    println!(
        "of the heap (Θ(Nk) against Θ(N log k)), and the heap needs {:.2}x those of the balanced tree",
        res[0].cmps as f64 / res[2].cmps as f64
    );

    // Same N, k = 32, three size shapes. writes = elements written by pairwise
    // merge steps (the sum of every merge's output length): the Huffman order
    // minimises it, and comparisons follow it closely.
    println!();
    println!(
        "(b) k = 32 lists, three size shapes (comparisons; then elements written by the merge steps)"
    );
    let even = vec![N / 32; 32];
    let geo: Vec<usize> = (0..32).map(|i| (N >> (i + 1)).max(1)).collect();
    let mut big = vec![100; 32];
    big[0] = N - 31 * 100;
    println!(
        "{:<10} {:>7} {:>8} {:>11} {:>9} {:>15} | {:>11} {:>9} {:>15}",
        "shape",
        "N",
        "heap",
        "sequential",
        "balanced",
        "smallest-first",
        "sequential",
        "balanced",
        "smallest-first"
    );
    let (mut fewest, mut skew_heap_wins) = (true, true);
    for (name, sizes) in [("even", &even), ("geometric", &geo), ("one-big", &big)] {
        res = run_all(&deal(&mut r, sizes, 1_000_000), &mut mismatches);
        fewest = fewest && res[3].writes <= res[1].writes && res[3].writes <= res[2].writes;
        if name != "even" {
            skew_heap_wins = skew_heap_wins && res[0].cmps < res[2].cmps;
        }
        println!(
            "{:<10} {:>7} {:>8} {:>11} {:>9} {:>15} | {:>11} {:>9} {:>15}",
            name,
            res[0].out.len(),
            res[0].cmps,
            res[1].cmps,
            res[2].cmps,
            res[3].cmps,
            res[1].writes,
            res[2].writes,
            res[3].writes
        );
    }
    println!(
        "smallest-first writes the fewest elements of the three pairwise orders in every shape: {fewest}"
    );
    println!(
        "on the two skewed shapes the heap is below balanced pairing, which ignores sizes: {skew_heap_wins}"
    );
    println!(
        "one-big: balanced {} comparisons, smallest-first {} ({:.1}x fewer), the heap {}",
        res[2].cmps,
        res[3].cmps,
        res[2].cmps as f64 / res[3].cmps as f64,
        res[0].cmps
    );

    // Laziness: heapify costs at most 2k comparisons and each output at most
    // 2⌊lg k⌋, so the first m values arrive after at most 2k + 2m⌊lg k⌋. The
    // eager strategies have produced nothing until they are done.
    println!();
    let k = lazy_lists.len();
    let lg = k.ilog2() as usize;
    let eager = run_all(&lazy_lists, &mut mismatches);
    let mut want: Vec<i64> = lazy_lists.concat();
    want.sort_unstable();
    println!(
        "(c) the first m outputs of the k = {k} heap merge (cumulative comparisons, heapify included)"
    );
    println!(
        "{:>9} {:>12} {:>16} {:>14}",
        "m", "comparisons", "bound 2k+2m⌊lg k⌋", "per output"
    );
    let mut m = KMerge::new(&lazy_lists);
    let (mut got, mut prefix_ok, mut bound_ok) = (0, true, true);
    println!("{:>9} {:>12} {:>16} {:>14}", 0, m.h.cmps, 2 * k, "-");
    for target in [1, 10, 100, 1000, 10_000, N] {
        // `by_ref().take(n)` hands out exactly the next n values, lazily
        for v in m.by_ref().take(target - got) {
            prefix_ok = prefix_ok && v == want[got];
            got += 1;
        }
        let bound = 2 * k + 2 * target * lg;
        bound_ok = bound_ok && m.h.cmps <= bound as u64;
        println!(
            "{:>9} {:>12} {:>16} {:>14.2}",
            target,
            m.h.cmps,
            bound,
            m.h.cmps as f64 / target as f64
        );
    }
    println!(
        "every prefix equals the sorted order: {prefix_ok};  every row within the bound: {bound_ok}"
    );
    let mut first = KMerge::new(&lazy_lists);
    first.next();
    println!(
        "the first output cost {} comparisons; balanced pairwise needs {} and the fold {} before any output exists",
        first.h.cmps, eager[2].cmps, eager[1].cmps
    );

    // Oracle: 2,000 tiny configurations, empty lists included, values IntN(10)
    // so there are many ties. Besides equality with concat+sort, Huffman's
    // order must write no more elements than either other pairwise order.
    println!();
    const CASES: u32 = 2000;
    let mut huffman_ok = true;
    for _ in 0..CASES {
        let count = 1 + r.int_n(8) as usize;
        let sizes: Vec<usize> = (0..count).map(|_| r.int_n(7) as usize).collect();
        res = run_all(&deal(&mut r, &sizes, 10), &mut mismatches);
        huffman_ok = huffman_ok && res[3].writes <= res[1].writes && res[3].writes <= res[2].writes;
    }
    println!(
        "(d) oracle: all four strategies, {CASES} tiny cases (k in 1..8, sizes 0..6, values IntN(10)) and the runs above"
    );
    println!(
        "mismatches: {mismatches};  smallest-first never writes more than balanced or the fold: {huffman_ok}"
    );

    // Rust only: the k = 64 lists of (a) through std's BinaryHeap. The own heap
    // is the same loops as Go's (and Go's container/heap, example 7); std has
    // its own sifts, so its counts are its own.
    println!();
    println!(
        "(e) the k = 64 lists of (a) through std::collections::BinaryHeap (comparisons between heads)"
    );
    let mut own = KMerge::new(&lists64);
    let own_out: Vec<i64> = own.by_ref().collect();
    let (out_peek, peek_cmps, peek_removal) = std_merge(&lists64, true);
    let (out_pop, pop_cmps, _) = std_merge(&lists64, false);
    println!("own heap, replace-root     {:>8}", own.h.cmps);
    println!(
        "BinaryHeap + peek_mut      {:>8}  (equal to own: {})",
        peek_cmps,
        peek_cmps == own.h.cmps
    );
    println!("BinaryHeap pop + push      {:>8}", pop_cmps);
    println!(
        "the {} list ends cost {} comparisons in the own heap and {} in PeekMut::pop; the other outputs replace the root in both",
        lists64.len(),
        own.removal_cmps,
        peek_removal
    );
    println!(
        "own - peek_mut = {}, removal difference = {}; all three outputs equal: {}",
        own.h.cmps - peek_cmps,
        own.removal_cmps - peek_removal,
        out_peek == own_out && out_pop == own_out
    );
}
```

**Rust output:**

```
(a) N = 100000 values IntN(1000000) dealt to k random lists: comparisons between elements
    k       heap  sequential  balanced  smallest-first    N·lg k   N⌈lg k⌉
    2      99999       99999     99999           99999    100000    100000
    4     249948      225092    199994          199995    200000    200000
   16     562923      839559    399965          399971    400000    400000
   64     921426     3248546    599873          599867    600000    600000
  256    1308188    12813802    799483          799479    800000    800000
 1024    1700121    50759014    997952          998045   1000000   1000000
balanced <= N⌈lg k⌉ in every row: true;  balanced <= heap in every row: true
the fold costs no more than the heap up to k = 4; at k = 1024 it needs 30x the comparisons
of the heap (Θ(Nk) against Θ(N log k)), and the heap needs 1.70x those of the balanced tree

(b) k = 32 lists, three size shapes (comparisons; then elements written by the merge steps)
shape            N     heap  sequential  balanced  smallest-first |  sequential  balanced  smallest-first
even        100000   737441     1646076    499934          499934 |     1646875    500000          500000
geometric   100010   394483     2093319    497765          200174 |     3050012    500050          200219
one-big     100000   223147     3033184    499432          115329 |     3053500    500000          115400
smallest-first writes the fewest elements of the three pairwise orders in every shape: true
on the two skewed shapes the heap is below balanced pairing, which ignores sizes: true
one-big: balanced 499432 comparisons, smallest-first 115329 (4.3x fewer), the heap 223147

(c) the first m outputs of the k = 1024 heap merge (cumulative comparisons, heapify included)
        m  comparisons bound 2k+2m⌊lg k⌋     per output
        0         1920             2048              -
        1         1938             2068        1938.00
       10         2092             2248         209.20
      100         3600             4048          36.00
     1000        18862            22048          18.86
    10000       171838           202048          17.18
   100000      1700121          2002048          17.00
every prefix equals the sorted order: true;  every row within the bound: true
the first output cost 1938 comparisons; balanced pairwise needs 997952 and the fold 50759014 before any output exists

(d) oracle: all four strategies, 2000 tiny cases (k in 1..8, sizes 0..6, values IntN(10)) and the runs above
mismatches: 0;  smallest-first never writes more than balanced or the fold: true

(e) the k = 64 lists of (a) through std::collections::BinaryHeap (comparisons between heads)
own heap, replace-root       921426
BinaryHeap + peek_mut        921259  (equal to own: false)
BinaryHeap pop + push        823318
the 64 list ends cost 471 comparisons in the own heap and 304 in PeekMut::pop; the other outputs replace the root in both
own - peek_mut = 167, removal difference = 167; all three outputs equal: true
```

> **Go vs Rust:** Parts (a) to (d) are identical to the Go run (diffed): the generic `Heap<T>` has the same loops, and the lazy merge is an `Iterator`, so "the first m outputs" is `.take(m)` with the cumulative count read between calls. In (e), std's `BinaryHeap` + `peek_mut` differs from the own heap by exactly the 64 list ends (167 comparisons: 304 in `PeekMut::pop` against 471), because `pop` is bottom-up with one comparison per level while the guard's drop is the textbook two-per-level sift (both read in the source, not run). Pop-then-push through std took 823,318 comparisons against 921,259 for `peek_mut`, and std needs a counting `Ord` wrapper (`StdHead`) where Go counts inside `lt`.

**Complexity:** heap Θ(N log k) comparisons and lazy: heapify at most 2k, then at most 2⌊lg k⌋ per output (1,938 comparisons for the first output at k = 1024) · balanced pairwise at most N⌈lg k⌉, eager (997,952 comparisons at k = 1024, against 1,700,121 for the heap) · the sequential fold Θ(Nk) (50,759,014 at k = 1024) · smallest-first minimises the elements written, and on the skewed shapes needed 2.5x (geometric) and 4.3x (one-big) fewer comparisons than balanced, while on equal sizes it ties it.

---

## 9. Running median with two heaps

`🟡 medium` · *Two heaps / order statistic*

A max-heap holds the lower half of everything seen and a min-heap the upper half; keep their sizes within one of each other and the top of the lower half at most the top of the upper half. Each insert is one push to the side the value belongs to plus at most one move across to restore the sizes, so it costs Θ(log n), and the median is read off the tops in Θ(1). That beats the sorted slice, which finds the slot by binary search but then shifts Θ(n) elements, Θ(n²) over a stream. The stream is integers, so the program prints **twice** the median (`2·lo.top` for an odd count, `lo.top + hi.top` for an even one) and never touches a float.

**Steps:**

1. Write one `Heap` with a `less` function, so the max-heap and the min-heap are the same code with `>` and `<`, and count its comparisons, pushes and pops.
2. Insert a stream while asserting both invariants after every insert, then run the same values in random, ascending, descending and all-equal order and count what an insert costs.
3. Compare the whole 2·median sequence with a sorted-slice oracle on 2,000 short streams full of ties and on the long stream, and count how many elements the oracle shifts.

```go
package main

import (
	"fmt"
	"math/bits"
	"math/rand/v2"
	"slices"
)

// Running median with two heaps. A MAX-heap `lo` holds the lower half of
// everything seen and a MIN-heap `hi` the upper half. Two invariants:
//
//	sizes:  len(lo) - len(hi) is 0 or 1   (so lo.top is the median when n is odd)
//	order:  lo.top <= hi.top              (every lower element <= every upper one)
//
// An insert is one push (to the side the value belongs to) and at most one move
// across to restore the sizes. The median is read in Theta(1): lo.top for an odd
// count, the mean of both tops for an even one. The stream is integers, so the
// program prints TWICE the median, 2*lo.top or lo.top+hi.top, and avoids floats.

// Heap is a binary heap ordered by lt, with the up/down loops of container/heap
// (strict lt, left child on ties, a swap at every level). Min and max are the
// same code with lt = < or lt = >. It counts comparisons, pushes and pops; the
// Rust twin ports the same loops, so every count below is identical.
type Heap struct {
	a                  []int
	lt                 func(x, y int) bool
	cmps, pushes, pops int
	popCmps            int // the part of cmps spent inside pop
}

func (h *Heap) less(i, j int) bool { h.cmps++; return h.lt(h.a[i], h.a[j]) }
func (h *Heap) swap(i, j int)      { h.a[i], h.a[j] = h.a[j], h.a[i] }

func (h *Heap) up(j int) {
	for {
		i := (j - 1) / 2
		if i == j || !h.less(j, i) {
			break
		}
		h.swap(i, j)
		j = i
	}
}

func (h *Heap) down(i, n int) {
	for {
		j1 := 2*i + 1
		if j1 >= n {
			break
		}
		j := j1
		if j2 := j1 + 1; j2 < n && h.less(j2, j1) {
			j = j2
		}
		if !h.less(j, i) {
			break
		}
		h.swap(i, j)
		i = j
	}
}

func (h *Heap) push(x int) { h.pushes++; h.a = append(h.a, x); h.up(len(h.a) - 1) }

func (h *Heap) pop() int {
	h.pops++
	before := h.cmps
	n := len(h.a) - 1
	h.swap(0, n)
	h.down(0, n)
	h.popCmps += h.cmps - before
	x := h.a[n]
	h.a = h.a[:n]
	return x
}

// isHeap checks the heap property directly (uncounted): no child orders before its parent.
func (h *Heap) isHeap() bool {
	for j := 1; j < len(h.a); j++ {
		if h.lt(h.a[j], h.a[(j-1)/2]) {
			return false
		}
	}
	return true
}

type Median struct {
	lo, hi         *Heap
	routing, moves int // routing comparisons x vs lo.top; moves across
}

func NewMedian() *Median {
	return &Median{
		lo: &Heap{lt: func(x, y int) bool { return x > y }},
		hi: &Heap{lt: func(x, y int) bool { return x < y }},
	}
}

// insert pushes x on the side it belongs to, then moves at most one element
// across. It returns which way a move went, for the trace.
func (m *Median) insert(x int) string {
	if len(m.lo.a) == 0 {
		m.lo.push(x)
	} else {
		m.routing++ // the one comparison that routes x
		if x <= m.lo.a[0] {
			m.lo.push(x)
		} else {
			m.hi.push(x)
		}
	}
	switch {
	case len(m.lo.a) > len(m.hi.a)+1:
		m.hi.push(m.lo.pop())
		m.moves++
		return "lo->hi"
	case len(m.hi.a) > len(m.lo.a):
		m.lo.push(m.hi.pop())
		m.moves++
		return "hi->lo"
	}
	return "-"
}

func (m *Median) twice() int {
	if len(m.lo.a) > len(m.hi.a) {
		return 2 * m.lo.a[0]
	}
	return m.lo.a[0] + m.hi.a[0]
}

// invariantsHold checks both invariants (sizes and order) after an insert.
func (m *Median) invariantsHold() bool {
	d := len(m.lo.a) - len(m.hi.a)
	return (d == 0 || d == 1) && (len(m.hi.a) == 0 || m.lo.a[0] <= m.hi.a[0])
}

func (m *Median) heapCmps() int { return m.lo.cmps + m.hi.cmps }
func (m *Median) popCmps() int  { return m.lo.popCmps + m.hi.popCmps }
func (m *Median) heapOps() int {
	return m.lo.pushes + m.hi.pushes + m.lo.pops + m.hi.pops
}

// oracle is the sorted-slice median: binary search, then insert, which shifts
// every element after the slot. That shifting is Θ(n²) over a stream.
type oracle struct {
	s      []int
	shifts int
}

func (o *oracle) insert(x int) {
	i, _ := slices.BinarySearch(o.s, x)
	o.shifts += len(o.s) - i
	o.s = slices.Insert(o.s, i, x)
}

func (o *oracle) twice() int {
	n := len(o.s)
	if n%2 == 1 {
		return 2 * o.s[n/2]
	}
	return o.s[n/2-1] + o.s[n/2]
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))

	fmt.Println("(a) the first 12 values of a stream, IntN(100): the heaps after each insert")
	fmt.Printf("%3s %4s %4s %4s %7s %7s %9s %7s\n", "n", "x", "|lo|", "|hi|", "lo.top", "hi.top", "2·median", "move")
	m := NewMedian()
	for i := 1; i <= 12; i++ {
		x := r.IntN(100)
		move := m.insert(x)
		hiTop := "-"
		if len(m.hi.a) > 0 {
			hiTop = fmt.Sprint(m.hi.a[0])
		}
		fmt.Printf("%3d %4d %4d %4d %7d %7s %9d %7s\n", i, x, len(m.lo.a), len(m.hi.a), m.lo.a[0], hiTop, m.twice(), move)
	}

	// 20,000 values, both invariants asserted after EVERY insert.
	const n = 20000
	stream := make([]int, n)
	for i := range stream {
		stream[i] = r.IntN(1000000)
	}
	fmt.Println()
	fmt.Printf("(b) a stream of %d values IntN(1000000), invariants checked after every insert\n", n)
	m = NewMedian()
	seq := make([]int, n)
	violations, maxOps := 0, 0
	fmt.Printf("%7s %7s %7s\n", "n", "|lo|", "|hi|")
	for i, x := range stream {
		before := m.heapOps()
		m.insert(x)
		maxOps = max(maxOps, m.heapOps()-before)
		seq[i] = m.twice()
		if !m.invariantsHold() {
			violations++
		}
		switch i + 1 {
		case 1, 2, 3, 1000, n - 1, n:
			fmt.Printf("%7d %7d %7d\n", i+1, len(m.lo.a), len(m.hi.a))
		}
	}
	fmt.Printf("invariant violations: %d\n", violations)
	pushes, pops := m.lo.pushes+m.hi.pushes, m.lo.pops+m.hi.pops
	fmt.Printf("pushes %d, pops %d, moves across %d (%.1f%% of inserts); pushes == n + moves: %v, pops == moves: %v\n",
		pushes, pops, m.moves, 100*float64(m.moves)/n, pushes == n+m.moves, pops == m.moves)
	fmt.Printf("heap operations per insert: at most %d (push, pop, push), %.2f on average\n",
		maxOps, float64(m.heapOps())/n)
	fmt.Printf("comparisons per insert: %.2f inside the heaps + %.2f routing = %.2f   (lg n = %.1f)\n",
		float64(m.heapCmps())/n, float64(m.routing)/n, float64(m.heapCmps()+m.routing)/n, float64(bits.Len(n)-1))

	fmt.Printf("of those %d heap comparisons, %d (%.0f%%) are inside the %d pops: %.1f per pop against %.2f per push\n",
		m.heapCmps(), m.popCmps(), 100*float64(m.popCmps())/float64(m.heapCmps()), pops,
		float64(m.popCmps())/float64(pops), float64(m.heapCmps()-m.popCmps())/float64(pushes))

	// The same structure on four input shapes: random order is the cheap case.
	fmt.Println()
	fmt.Printf("(b2) the same %d values in four orders: what the structure pays per insert\n", n)
	fmt.Printf("%-11s %8s %12s %14s %8s\n", "shape", "moves", "cmps/insert", "routing/insert", "max ops")
	sorted := slices.Sorted(slices.Values(stream))
	desc := slices.Clone(sorted)
	slices.Reverse(desc)
	for _, sh := range []struct {
		name string
		xs   []int
	}{{"random", stream}, {"ascending", sorted}, {"descending", desc}, {"all equal", make([]int, n)}} {
		mm := NewMedian()
		mo := 0
		for _, x := range sh.xs {
			before := mm.heapOps()
			mm.insert(x)
			mo = max(mo, mm.heapOps()-before)
		}
		fmt.Printf("%-11s %8d %12.2f %14.2f %8d\n", sh.name, mm.moves,
			float64(mm.heapCmps())/n, float64(mm.routing)/n, mo)
	}

	// Oracle: a sorted slice kept in order. 2,000 short streams with many ties
	// (values IntN(20)) and the long stream above, compared at every insert.
	fmt.Println()
	mismatches, heapViolations, checked := 0, 0, 0
	for range 2000 {
		mm, o := NewMedian(), &oracle{}
		for range 1 + r.IntN(300) {
			x := r.IntN(20)
			mm.insert(x)
			o.insert(x)
			checked++
			if mm.twice() != o.twice() {
				mismatches++
			}
			if !mm.invariantsHold() {
				violations++
			}
			if !mm.lo.isHeap() || !mm.hi.isHeap() {
				heapViolations++
			}
		}
	}
	o := &oracle{}
	for i, x := range stream {
		o.insert(x)
		checked++
		if seq[i] != o.twice() {
			mismatches++
		}
	}
	fmt.Printf("(c) oracle: 2000 streams (length 1..300, values IntN(20)) and the long stream, %d inserts compared\n", checked)
	fmt.Printf("mismatches: %d  invariant violations: %d  heap-order violations: %d\n", mismatches, violations, heapViolations)
	fmt.Printf("on the long stream the sorted slice shifted %d elements; the two heaps made %d comparisons and %d moves\n",
		o.shifts, m.heapCmps()+m.routing, m.moves)
}
```

**Output:**

```
(a) the first 12 values of a stream, IntN(100): the heaps after each insert
  n    x |lo| |hi|  lo.top  hi.top  2·median    move
  1   76    1    0      76       -       152       -
  2   61    1    1      61      76       137  lo->hi
  3   78    2    1      76      78       152  hi->lo
  4   79    2    2      76      78       154       -
  5   23    3    2      76      78       152       -
  6    4    3    3      61      76       137  lo->hi
  7   49    4    3      61      76       122       -
  8   44    4    4      49      61       110  lo->hi
  9   13    5    4      49      61        98       -
 10   18    5    5      44      49        93  lo->hi
 11   66    6    5      49      61        98  hi->lo
 12   44    6    6      44      49        93  lo->hi

(b) a stream of 20000 values IntN(1000000), invariants checked after every insert
      n    |lo|    |hi|
      1       1       0
      2       1       1
      3       2       1
   1000     500     500
  19999   10000    9999
  20000   10000   10000
invariant violations: 0
pushes 29989, pops 9989, moves across 9989 (49.9% of inserts); pushes == n + moves: true, pops == moves: true
heap operations per insert: at most 3 (push, pop, push), 2.00 on average
comparisons per insert: 18.79 inside the heaps + 1.00 routing = 19.79   (lg n = 14.0)
of those 375735 heap comparisons, 222242 (59%) are inside the 9989 pops: 22.2 per pop against 5.12 per push

(b2) the same 20000 values in four orders: what the structure pays per insert
shape          moves  cmps/insert routing/insert  max ops
random          9989        18.79           1.00        3
ascending       9999        17.19           1.00        3
descending     10000        17.19           1.00        3
all equal      10000         2.50           1.00        3

(c) oracle: 2000 streams (length 1..300, values IntN(20)) and the long stream, 324636 inserts compared
mismatches: 0  invariant violations: 0  heap-order violations: 0
on the long stream the sorted slice shifted 100371179 elements; the two heaps made 395734 comparisons and 9989 moves
```

**Rust:**

```rust
use std::cell::Cell;
use std::cmp::{Ordering, Reverse};
use std::collections::BinaryHeap;

// Running median with two heaps. A MAX-heap `lo` holds the lower half of
// everything seen and a MIN-heap `hi` the upper half. Two invariants:
//
//     sizes:  len(lo) - len(hi) is 0 or 1   (so lo.top is the median when n is odd)
//     order:  lo.top <= hi.top              (every lower element <= every upper one)
//
// An insert is one push (to the side the value belongs to) and at most one move
// across to restore the sizes. The median is read in Theta(1): lo.top for an odd
// count, the mean of both tops for an even one. The stream is integers, so the
// program prints TWICE the median, 2*lo.top or lo.top+hi.top, and avoids floats.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and of its IntN (see lesson
// 01, example 14), so both programs draw the same stream and the same counts
// come out.
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

// Heap is a binary heap ordered by `lt`, with a verbatim port of the up/down
// loops of Go's container/heap (strict lt, left child on ties, a swap at every
// level). Min and max are the same code with lt = < or lt = >. It counts
// comparisons, pushes and pops, exactly as the Go program does.
struct Heap {
    a: Vec<i64>,
    lt: fn(i64, i64) -> bool,
    cmps: u64,
    pushes: u64,
    pops: u64,
    pop_cmps: u64, // the part of cmps spent inside pop
}

impl Heap {
    fn new(lt: fn(i64, i64) -> bool) -> Self {
        Heap {
            a: Vec::new(),
            lt,
            cmps: 0,
            pushes: 0,
            pops: 0,
            pop_cmps: 0,
        }
    }

    fn less(&mut self, i: usize, j: usize) -> bool {
        self.cmps += 1;
        (self.lt)(self.a[i], self.a[j])
    }

    // A usize has no -1, so the loop condition is `j > 0` where Go tests i == j.
    fn up(&mut self, mut j: usize) {
        while j > 0 {
            let i = (j - 1) / 2;
            if !self.less(j, i) {
                break;
            }
            self.a.swap(i, j);
            j = i;
        }
    }

    fn down(&mut self, mut i: usize, n: usize) {
        loop {
            let j1 = 2 * i + 1;
            if j1 >= n {
                break;
            }
            let mut j = j1;
            let j2 = j1 + 1;
            if j2 < n && self.less(j2, j1) {
                j = j2;
            }
            if !self.less(j, i) {
                break;
            }
            self.a.swap(i, j);
            i = j;
        }
    }

    fn push(&mut self, x: i64) {
        self.pushes += 1;
        self.a.push(x);
        self.up(self.a.len() - 1);
    }

    fn pop(&mut self) -> i64 {
        self.pops += 1;
        let before = self.cmps;
        let n = self.a.len() - 1;
        self.a.swap(0, n);
        self.down(0, n);
        self.pop_cmps += self.cmps - before;
        self.a.pop().expect("pop on a non-empty heap")
    }

    // is_heap checks the heap property directly (uncounted): no child orders before its parent.
    fn is_heap(&self) -> bool {
        (1..self.a.len()).all(|j| !(self.lt)(self.a[j], self.a[(j - 1) / 2]))
    }
}

struct Median {
    lo: Heap,
    hi: Heap,
    routing: u64, // routing comparisons x vs lo.top
    moves: u64,   // moves across
}

impl Median {
    fn new() -> Self {
        Median {
            lo: Heap::new(|x, y| x > y),
            hi: Heap::new(|x, y| x < y),
            routing: 0,
            moves: 0,
        }
    }

    // insert pushes x on the side it belongs to, then moves at most one element
    // across. It returns which way a move went, for the trace.
    fn insert(&mut self, x: i64) -> &'static str {
        if self.lo.a.is_empty() {
            self.lo.push(x);
        } else {
            self.routing += 1; // the one comparison that routes x
            if x <= self.lo.a[0] {
                self.lo.push(x);
            } else {
                self.hi.push(x);
            }
        }
        if self.lo.a.len() > self.hi.a.len() + 1 {
            let v = self.lo.pop();
            self.hi.push(v);
            self.moves += 1;
            "lo->hi"
        } else if self.hi.a.len() > self.lo.a.len() {
            let v = self.hi.pop();
            self.lo.push(v);
            self.moves += 1;
            "hi->lo"
        } else {
            "-"
        }
    }

    fn twice(&self) -> i64 {
        if self.lo.a.len() > self.hi.a.len() {
            2 * self.lo.a[0]
        } else {
            self.lo.a[0] + self.hi.a[0]
        }
    }

    // invariants_hold checks both invariants (sizes and order) after an insert.
    fn invariants_hold(&self) -> bool {
        let d = self.lo.a.len() as i64 - self.hi.a.len() as i64;
        (d == 0 || d == 1) && (self.hi.a.is_empty() || self.lo.a[0] <= self.hi.a[0])
    }

    fn heap_cmps(&self) -> u64 {
        self.lo.cmps + self.hi.cmps
    }

    fn pop_cmps(&self) -> u64 {
        self.lo.pop_cmps + self.hi.pop_cmps
    }

    fn heap_ops(&self) -> u64 {
        self.lo.pushes + self.hi.pushes + self.lo.pops + self.hi.pops
    }
}

// Oracle is the sorted-slice median: binary search, then insert, which shifts
// every element after the slot. That shifting is Θ(n²) over a stream.
// `partition_point` gives the leftmost slot, like Go's slices.BinarySearch;
// Rust's `binary_search` may return any of several equal elements' slots.
struct Oracle {
    s: Vec<i64>,
    shifts: u64,
}

impl Oracle {
    fn insert(&mut self, x: i64) {
        let i = self.s.partition_point(|&v| v < x);
        self.shifts += (self.s.len() - i) as u64;
        self.s.insert(i, x);
    }

    fn twice(&self) -> i64 {
        let n = self.s.len();
        if n % 2 == 1 {
            2 * self.s[n / 2]
        } else {
            self.s[n / 2 - 1] + self.s[n / 2]
        }
    }
}

thread_local!(static COMPARES: Cell<u64> = const { Cell::new(0) });

// Counted is the element type for the standard library's heaps: its `Ord::cmp`
// bumps a counter, and every comparison operator BinaryHeap uses (`<=`, `>=`)
// goes through `partial_cmp`, which calls `cmp` once.
#[derive(PartialEq, Eq)]
struct Counted(i64);

impl Ord for Counted {
    fn cmp(&self, other: &Self) -> Ordering {
        COMPARES.with(|c| c.set(c.get() + 1));
        self.0.cmp(&other.0)
    }
}

impl PartialOrd for Counted {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

// std_median: the same two-heap algorithm on BinaryHeap<Counted> (a max-heap,
// the lower half) and BinaryHeap<Reverse<Counted>> (a min-heap, the upper
// half). It returns the 2·median after every insert and the comparisons made.
fn std_median(stream: &[i64]) -> (Vec<i64>, u64, u64) {
    COMPARES.with(|c| c.set(0));
    let mut lo: BinaryHeap<Counted> = BinaryHeap::new();
    let mut hi: BinaryHeap<Reverse<Counted>> = BinaryHeap::new();
    let mut seq = Vec::with_capacity(stream.len());
    let mut pop_cmps = 0; // comparisons spent inside the two pops
    for &x in stream {
        // routing: a comparison only when lo is not empty
        if lo.peek().is_none_or(|top| Counted(x) <= *top) {
            lo.push(Counted(x));
        } else {
            hi.push(Reverse(Counted(x)));
        }
        if lo.len() > hi.len() + 1 {
            let before = COMPARES.with(Cell::get);
            let v = lo.pop().expect("lo is not empty");
            pop_cmps += COMPARES.with(Cell::get) - before;
            hi.push(Reverse(v));
        } else if hi.len() > lo.len() {
            let before = COMPARES.with(Cell::get);
            let Reverse(v) = hi.pop().expect("hi is not empty");
            pop_cmps += COMPARES.with(Cell::get) - before;
            lo.push(v);
        }
        let lo_top = lo.peek().expect("an insert was made").0;
        seq.push(if lo.len() > hi.len() {
            2 * lo_top
        } else {
            lo_top + hi.peek().expect("hi is not empty").0.0
        });
    }
    (seq, COMPARES.with(Cell::get), pop_cmps)
}

fn main() {
    let mut r = Pcg::new(1, 2);

    println!("(a) the first 12 values of a stream, IntN(100): the heaps after each insert");
    println!(
        "{:>3} {:>4} {:>4} {:>4} {:>7} {:>7} {:>9} {:>7}",
        "n", "x", "|lo|", "|hi|", "lo.top", "hi.top", "2·median", "move"
    );
    let mut m = Median::new();
    for i in 1..=12 {
        let x = r.int_n(100) as i64;
        let mv = m.insert(x);
        let hi_top =
            m.hi.a
                .first()
                .map_or_else(|| "-".to_string(), i64::to_string);
        println!(
            "{:>3} {:>4} {:>4} {:>4} {:>7} {:>7} {:>9} {:>7}",
            i,
            x,
            m.lo.a.len(),
            m.hi.a.len(),
            m.lo.a[0],
            hi_top,
            m.twice(),
            mv
        );
    }

    // 20,000 values, both invariants asserted after EVERY insert.
    const N: usize = 20000;
    let nf = N as f64;
    let stream: Vec<i64> = (0..N).map(|_| r.int_n(1_000_000) as i64).collect();
    println!();
    println!("(b) a stream of {N} values IntN(1000000), invariants checked after every insert");
    m = Median::new();
    let mut seq = Vec::with_capacity(N);
    let (mut violations, mut max_ops) = (0u32, 0u64);
    println!("{:>7} {:>7} {:>7}", "n", "|lo|", "|hi|");
    for (i, &x) in stream.iter().enumerate() {
        let before = m.heap_ops();
        m.insert(x);
        max_ops = max_ops.max(m.heap_ops() - before);
        seq.push(m.twice());
        if !m.invariants_hold() {
            violations += 1;
        }
        if matches!(i + 1, 1 | 2 | 3 | 1000 | 19999 | 20000) {
            println!("{:>7} {:>7} {:>7}", i + 1, m.lo.a.len(), m.hi.a.len());
        }
    }
    println!("invariant violations: {violations}");
    let (pushes, pops) = (m.lo.pushes + m.hi.pushes, m.lo.pops + m.hi.pops);
    println!(
        "pushes {pushes}, pops {pops}, moves across {} ({:.1}% of inserts); pushes == n + moves: {}, pops == moves: {}",
        m.moves,
        100.0 * m.moves as f64 / nf,
        pushes == N as u64 + m.moves,
        pops == m.moves
    );
    println!(
        "heap operations per insert: at most {max_ops} (push, pop, push), {:.2} on average",
        m.heap_ops() as f64 / nf
    );
    println!(
        "comparisons per insert: {:.2} inside the heaps + {:.2} routing = {:.2}   (lg n = {:.1})",
        m.heap_cmps() as f64 / nf,
        m.routing as f64 / nf,
        (m.heap_cmps() + m.routing) as f64 / nf,
        N.ilog2() as f64
    );

    println!(
        "of those {} heap comparisons, {} ({:.0}%) are inside the {} pops: {:.1} per pop against {:.2} per push",
        m.heap_cmps(),
        m.pop_cmps(),
        100.0 * m.pop_cmps() as f64 / m.heap_cmps() as f64,
        pops,
        m.pop_cmps() as f64 / pops as f64,
        (m.heap_cmps() - m.pop_cmps()) as f64 / pushes as f64
    );

    // The same structure on four input shapes: random order is the cheap case.
    println!();
    println!("(b2) the same {N} values in four orders: what the structure pays per insert");
    println!(
        "{:<11} {:>8} {:>12} {:>14} {:>8}",
        "shape", "moves", "cmps/insert", "routing/insert", "max ops"
    );
    let mut sorted = stream.clone();
    sorted.sort_unstable();
    let desc: Vec<i64> = sorted.iter().rev().copied().collect();
    let equal = vec![0i64; N];
    for (name, xs) in [
        ("random", &stream),
        ("ascending", &sorted),
        ("descending", &desc),
        ("all equal", &equal),
    ] {
        let mut mm = Median::new();
        let mut mo = 0;
        for &x in xs {
            let before = mm.heap_ops();
            mm.insert(x);
            mo = mo.max(mm.heap_ops() - before);
        }
        println!(
            "{:<11} {:>8} {:>12.2} {:>14.2} {:>8}",
            name,
            mm.moves,
            mm.heap_cmps() as f64 / nf,
            mm.routing as f64 / nf,
            mo
        );
    }

    // Oracle: a sorted slice kept in order. 2,000 short streams with many ties
    // (values IntN(20)) and the long stream above, compared at every insert.
    println!();
    let (mut mismatches, mut heap_violations, mut checked) = (0u32, 0u32, 0u32);
    for _ in 0..2000 {
        let mut mm = Median::new();
        let mut o = Oracle {
            s: Vec::new(),
            shifts: 0,
        };
        for _ in 0..1 + r.int_n(300) {
            let x = r.int_n(20) as i64;
            mm.insert(x);
            o.insert(x);
            checked += 1;
            if mm.twice() != o.twice() {
                mismatches += 1;
            }
            if !mm.invariants_hold() {
                violations += 1;
            }
            if !mm.lo.is_heap() || !mm.hi.is_heap() {
                heap_violations += 1;
            }
        }
    }
    let mut o = Oracle {
        s: Vec::new(),
        shifts: 0,
    };
    for (i, &x) in stream.iter().enumerate() {
        o.insert(x);
        checked += 1;
        if seq[i] != o.twice() {
            mismatches += 1;
        }
    }
    println!(
        "(c) oracle: 2000 streams (length 1..300, values IntN(20)) and the long stream, {checked} inserts compared"
    );
    println!(
        "mismatches: {mismatches}  invariant violations: {violations}  heap-order violations: {heap_violations}"
    );
    println!(
        "on the long stream the sorted slice shifted {} elements; the two heaps made {} comparisons and {} moves",
        o.shifts,
        m.heap_cmps() + m.routing,
        m.moves
    );

    // Rust only: the same algorithm on std's BinaryHeap. The own heap is the
    // same loops as Go's; std has its own sifts, so its count is its own.
    println!();
    println!(
        "(d) the long stream through std::collections::BinaryHeap (max-heap lo, Reverse min-heap hi)"
    );
    let (std_seq, std_cmps, std_pop_cmps) = std_median(&stream);
    println!(
        "2·median sequence equal to the own heaps' at all {N} inserts: {}",
        std_seq == seq
    );
    println!(
        "comparisons: own heaps {} (inside the heaps + routing), std {}",
        m.heap_cmps() + m.routing,
        std_cmps
    );
    println!(
        "inside the pops: own {}, std {};  everything else: own {}, std {}",
        m.pop_cmps(),
        std_pop_cmps,
        m.heap_cmps() + m.routing - m.pop_cmps(),
        std_cmps - std_pop_cmps
    );
}
```

**Rust output:**

```
(a) the first 12 values of a stream, IntN(100): the heaps after each insert
  n    x |lo| |hi|  lo.top  hi.top  2·median    move
  1   76    1    0      76       -       152       -
  2   61    1    1      61      76       137  lo->hi
  3   78    2    1      76      78       152  hi->lo
  4   79    2    2      76      78       154       -
  5   23    3    2      76      78       152       -
  6    4    3    3      61      76       137  lo->hi
  7   49    4    3      61      76       122       -
  8   44    4    4      49      61       110  lo->hi
  9   13    5    4      49      61        98       -
 10   18    5    5      44      49        93  lo->hi
 11   66    6    5      49      61        98  hi->lo
 12   44    6    6      44      49        93  lo->hi

(b) a stream of 20000 values IntN(1000000), invariants checked after every insert
      n    |lo|    |hi|
      1       1       0
      2       1       1
      3       2       1
   1000     500     500
  19999   10000    9999
  20000   10000   10000
invariant violations: 0
pushes 29989, pops 9989, moves across 9989 (49.9% of inserts); pushes == n + moves: true, pops == moves: true
heap operations per insert: at most 3 (push, pop, push), 2.00 on average
comparisons per insert: 18.79 inside the heaps + 1.00 routing = 19.79   (lg n = 14.0)
of those 375735 heap comparisons, 222242 (59%) are inside the 9989 pops: 22.2 per pop against 5.12 per push

(b2) the same 20000 values in four orders: what the structure pays per insert
shape          moves  cmps/insert routing/insert  max ops
random          9989        18.79           1.00        3
ascending       9999        17.19           1.00        3
descending     10000        17.19           1.00        3
all equal      10000         2.50           1.00        3

(c) oracle: 2000 streams (length 1..300, values IntN(20)) and the long stream, 324636 inserts compared
mismatches: 0  invariant violations: 0  heap-order violations: 0
on the long stream the sorted slice shifted 100371179 elements; the two heaps made 395734 comparisons and 9989 moves

(d) the long stream through std::collections::BinaryHeap (max-heap lo, Reverse min-heap hi)
2·median sequence equal to the own heaps' at all 20000 inserts: true
comparisons: own heaps 395734 (inside the heaps + routing), std 296166
inside the pops: own 222242, std 122655;  everything else: own 173492, std 173511
```

> **Go vs Rust:** Parts (a) to (c) are identical to the Go run (diffed); the one `Heap` takes a `fn(i64, i64) -> bool`, so the max-heap is a non-capturing closure `|x, y| x > y`, and the oracle uses `partition_point` for the leftmost slot because `binary_search` may return any of several equal elements' slots (Go's `slices.BinarySearch` returns the leftmost). In (d), `BinaryHeap<Counted>` and `BinaryHeap<Reverse<Counted>>` give the same 2·median sequence with 296,166 comparisons against the own heaps' 395,734, and the difference sits in the 9,989 pops (122,655 against 222,242 comparisons), where std's `pop` is bottom-up with one comparison per level (read in the source, not run). Everything outside the pops differs by only 19 comparisons here (tied keys; in a separate run with all keys made distinct it was exactly equal).

**Complexity:** insert Θ(log n) (one push and at most one pop and push: at most 3 heap operations, 2.00 on average), median Θ(1), space Θ(n). On the 20,000-value random stream an insert cost 18.79 comparisons inside the heaps plus 1 routing comparison, about 1.4 lg n, of which 59% were spent inside the pops; every size and order invariant held after every insert. The sorted-slice oracle shifted 100,371,179 elements for the same stream where the two heaps made 395,734 comparisons and 9,989 moves.

---

## 10. Scheduling: meeting rooms and cooldowns

`🟡 medium` · *Greedy with a priority queue*

"Of what I can do now, which matters most?" is a priority queue, and two classic questions are exactly that. **Meeting rooms:** sort by start and keep the end time of every room in use in a min-heap; the root is the room that frees first, so each meeting either takes it over (pop, push) or opens a new one (push), and the heap's final size is the number of rooms. **Task scheduler with a cooldown:** a max-heap of remaining counts picks the most urgent letter each tick, and a FIFO holds the letters that are cooling down (their ready times come out already sorted). Both have exact oracles, a sweep over sorted starts and ends, the definition "most meetings containing one instant", a closed form for the cooldown length and an exhaustive search of every schedule, so both are proved rather than trusted. Rooms cost Θ(n log n) (the sort; the heap holds at most n end times) and the scheduler Θ(T log 26), which is Θ(T) for T tasks.

**Steps:**

1. Build the room heap and trace six meetings (the heap of end times after each step), then run 2,000 random cases against both oracles, plus the wrong rule that treats `end == start` as a clash.
2. Build the scheduler (max-heap on `(count, letter)` so the order is strict, FIFO of cooling tasks), trace `AAABBBC` with n = 2 tick by tick, then check 20,000 random cases against the closed form `max(T, (maxCount-1)*(n+1) + numberOfMaxCount)` and 300 tiny ones against an exhaustive search.
3. Read the counts: 0 mismatches everywhere, how often ties and idle ticks actually occur, and on 1,000 tasks one pop per task from a heap that never holds more than 26 letters.

```go
package main

import (
	"container/heap"
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
)

// "Of the things I can do right now, which matters most?" is a priority queue.
// Two classic shapes, each proved against oracles instead of trusted:
//
//	(a) meeting rooms:  a min-heap of END times; its size is the rooms in use
//	(b) task cooldown:  a max-heap of remaining counts + a FIFO of cooling tasks

// ---------------------------------------------------------------- (a) rooms

type meeting struct{ start, end int } // half-open: [start, end)

// minInts is the min-heap container/heap wants: Len/Less/Swap plus Push/Pop.
type minInts []int

func (h minInts) Len() int           { return len(h) }
func (h minInts) Less(i, j int) bool { return h[i] < h[j] }
func (h minInts) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *minInts) Push(x any)        { *h = append(*h, x.(int)) }
func (h *minInts) Pop() any {
	old := *h
	x := old[len(old)-1]
	*h = old[:len(old)-1]
	return x
}

// rooms sorts by start, then keeps the end time of every room in use in a heap.
// The root is the room that frees up first: if it is free by this meeting's
// start, reuse it (pop, push the new end); otherwise open a room (push).
// touching says whether end == start still counts as a clash (a closed
// interval); the real rule for [start, end) is that it does not.
func rooms(ms []meeting, touching bool, trace func(m meeting, note string, h minInts)) int {
	sorted := slices.Clone(ms)
	slices.SortFunc(sorted, func(a, b meeting) int {
		if a.start != b.start {
			return a.start - b.start
		}
		return a.end - b.end
	})
	h := &minInts{}
	for _, m := range sorted {
		note := "new room"
		if h.Len() > 0 && ((*h)[0] < m.start || (!touching && (*h)[0] == m.start)) {
			note = fmt.Sprintf("reuse the room freed at %d", heap.Pop(h))
		}
		heap.Push(h, m.end)
		if trace != nil {
			trace(m, note, *h)
		}
	}
	return h.Len() // it never shrinks below the number of rooms ever needed
}

// Oracle 1: sweep two sorted lists. Every start either reuses the earliest
// unmatched end (ends[j] <= start) or needs a new room.
func roomsSweep(ms []meeting) int {
	starts := make([]int, len(ms))
	ends := make([]int, len(ms))
	for i, m := range ms {
		starts[i], ends[i] = m.start, m.end
	}
	slices.Sort(starts)
	slices.Sort(ends)
	need, j := 0, 0
	for _, s := range starts {
		if ends[j] <= s {
			j++
		} else {
			need++
		}
	}
	return need
}

// Oracle 2: the definition. Rooms needed = the most meetings that contain a
// single instant t (start <= t < end). Obviously right, O(n * horizon).
func roomsByInstant(ms []meeting) int {
	best := 0
	for t := 0; t < 64; t++ {
		in := 0
		for _, m := range ms {
			if m.start <= t && t < m.end {
				in++
			}
		}
		best = max(best, in)
	}
	return best
}

// ------------------------------------------------------------- (b) cooldown

// A task type: how many runs remain, and its letter. Max-heap on count; ties
// go to the smaller letter, so the order is strict and the schedule is unique.
type task struct {
	count  int
	letter byte
}

func before(a, b task) bool {
	if a.count != b.count {
		return a.count > b.count
	}
	return a.letter < b.letter
}

type taskHeap []task

func (h taskHeap) Len() int           { return len(h) }
func (h taskHeap) Less(i, j int) bool { return before(h[i], h[j]) }
func (h taskHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *taskHeap) Push(x any)        { *h = append(*h, x.(task)) }
func (h *taskHeap) Pop() any {
	old := *h
	x := old[len(old)-1]
	*h = old[:len(old)-1]
	return x
}

// cooling is a task that ran and may run again at tick ready.
type cooling struct {
	ready int
	t     task
}

type stats struct{ pushes, pops, maxHeap int }

// schedule runs one tick at a time: first let tasks whose cooldown is over back
// into the heap, then run the most urgent one, or idle. Ready times are
// t+n+1 with t increasing, so a plain FIFO is already sorted by ready time.
func schedule(counts []int, n int, trace func(tick int, h taskHeap, q []cooling, ran string)) (string, stats) {
	var st stats
	h := &taskHeap{}
	for i, c := range counts {
		if c > 0 {
			heap.Push(h, task{c, byte('A' + i)})
			st.pushes++
		}
	}
	var q []cooling
	var out strings.Builder
	for tick := 0; h.Len() > 0 || len(q) > 0; tick++ {
		st.maxHeap = max(st.maxHeap, h.Len())
		for len(q) > 0 && q[0].ready <= tick {
			heap.Push(h, q[0].t)
			st.pushes++
			q = q[1:]
		}
		st.maxHeap = max(st.maxHeap, h.Len())
		ran := "_"
		if h.Len() > 0 {
			t := heap.Pop(h).(task)
			st.pops++
			ran = string(t.letter)
			if t.count > 1 {
				q = append(q, cooling{tick + n + 1, task{t.count - 1, t.letter}})
			}
		}
		out.WriteString(ran)
		if trace != nil {
			trace(tick, *h, q, ran)
		}
	}
	return out.String(), st
}

// closedForm: the busiest letter forces (maxCount-1) full frames of n+1 slots
// and a last partial frame with one slot per letter tied for the maximum;
// if there are more tasks than that, no idle slot is ever needed.
func closedForm(counts []int, n int) int {
	total, maxC, numMax := 0, 0, 0
	for _, c := range counts {
		total += c
		if c > maxC {
			maxC, numMax = c, 1
		} else if c == maxC {
			numMax++
		}
	}
	if total == 0 {
		return 0
	}
	return max(total, (maxC-1)*(n+1)+numMax)
}

// validSchedule checks the three rules independently of the simulation: every
// task type is run exactly its count, and two runs of one letter are at least
// n+1 ticks apart.
func validSchedule(s string, counts []int, n int) bool {
	last := make(map[byte]int)
	ran := make(map[byte]int)
	for i := 0; i < len(s); i++ {
		c := s[i]
		if c == '_' {
			continue
		}
		if p, ok := last[c]; ok && i-p < n+1 {
			return false
		}
		last[c] = i
		ran[c]++
	}
	for i, c := range counts {
		if ran[byte('A'+i)] != c {
			return false
		}
	}
	return true
}

// exhaustive tries EVERY schedule (any letter that is allowed, or idle, at
// every tick) and returns the shortest length. It prunes a branch only when it
// cannot beat the best so far: t ticks used + remaining tasks >= best.
func exhaustive(counts []int, n int) (best, nodes int) {
	rem := slices.Clone(counts)
	nextOK := make([]int, len(counts))
	left := 0
	for _, c := range counts {
		left += c
	}
	best = left*(n+1) + 1 // a feasible length plus one: anything real beats it
	var dfs func(t, left int)
	dfs = func(t, left int) {
		nodes++
		if left == 0 {
			best = min(best, t)
			return
		}
		if t+left >= best {
			return
		}
		for i := range rem {
			if rem[i] > 0 && nextOK[i] <= t {
				saved := nextOK[i]
				rem[i]--
				nextOK[i] = t + n + 1
				dfs(t+1, left-1)
				rem[i]++
				nextOK[i] = saved
			}
		}
		dfs(t+1, left) // idle, even when something could run
	}
	dfs(0, left)
	return best, nodes
}

func heapString(h taskHeap) string {
	s := slices.Clone(h)
	slices.SortFunc(s, func(a, b task) int {
		switch {
		case before(a, b):
			return -1
		case before(b, a):
			return 1
		}
		return 0
	})
	parts := make([]string, len(s))
	for i, t := range s {
		parts[i] = fmt.Sprintf("%c%d", t.letter, t.count)
	}
	return strings.Join(parts, " ")
}

func queueString(q []cooling) string {
	parts := make([]string, len(q))
	for i, c := range q {
		parts[i] = fmt.Sprintf("%c%d@%d", c.t.letter, c.t.count, c.ready)
	}
	return strings.Join(parts, " ")
}

func countsOf(word string) []int {
	counts := make([]int, 26)
	for i := 0; i < len(word); i++ {
		counts[word[i]-'A']++
	}
	return counts
}

func main() {
	fmt.Println("== (a) meeting rooms: a min-heap of end times ==")
	fmt.Println()
	ms := []meeting{{0, 5}, {1, 4}, {3, 9}, {4, 6}, {9, 12}, {10, 11}}
	fmt.Println("meetings [start,end):  A[0,5) B[1,4) C[3,9) D[4,6) E[9,12) F[10,11)")
	fmt.Printf("  %-9s %-30s %s\n", "meeting", "what happened", "heap of end times after (sorted)")
	names := "ABCDEF"
	k := 0
	need := rooms(ms, false, func(m meeting, note string, h minInts) {
		label := fmt.Sprintf("%c[%d,%d)", names[k], m.start, m.end)
		fmt.Printf("  %-9s %-30s %v\n", label, note, slices.Sorted(slices.Values(h)))
		k++
	})
	fmt.Printf("rooms needed: %d (the final heap size)\n", need)
	fmt.Println("D starts at 4 exactly when B ends, and takes B's room: [start,end) is half-open")

	r := rand.New(rand.NewPCG(1, 2))
	const cases = 2000
	var mism1, mism2, closedDiffers, total, widest int
	hist := make(map[int]int)
	for range cases {
		ms := make([]meeting, r.IntN(41))
		for i := range ms {
			s := r.IntN(30)
			ms[i] = meeting{s, s + 1 + r.IntN(8)}
		}
		got := rooms(ms, false, nil)
		if got != roomsSweep(ms) {
			mism1++
		}
		if got != roomsByInstant(ms) {
			mism2++
		}
		if rooms(ms, true, nil) != got {
			closedDiffers++
		}
		total += len(ms)
		widest = max(widest, got)
		hist[got]++
	}
	fmt.Println()
	fmt.Printf("%d random cases (%d meetings in all, start 0..29, length 1..8, so ties are common)\n", cases, total)
	fmt.Printf("mismatches against the sorted-starts/sorted-ends sweep: %d\n", mism1)
	fmt.Printf("mismatches against \"most meetings containing one instant\": %d\n", mism2)
	fmt.Printf("busiest case needed %d rooms; cases needing 0..%d rooms: ", widest, widest)
	for w := 0; w <= widest; w++ {
		if w > 0 {
			fmt.Print(" ")
		}
		fmt.Print(hist[w])
	}
	fmt.Println()
	fmt.Printf("treating end == start as a clash (closed intervals) gives a different answer in %d of %d cases\n", closedDiffers, cases)

	fmt.Println()
	fmt.Println("== (b) task scheduler with a cooldown: a max-heap + a FIFO of cooling tasks ==")
	fmt.Println()
	s, _ := schedule(countsOf("AAABBB"), 2, nil)
	fmt.Printf("AAABBB, n=2 -> %s (length %d, closed form %d)\n", s, len(s), closedForm(countsOf("AAABBB"), 2))
	s, _ = schedule(countsOf("AAABBBC"), 2, func(tick int, h taskHeap, q []cooling, ran string) {
		if tick == 0 {
			fmt.Println("AAABBBC, n=2, tick by tick (heap and cooling shown after the tick; a cooling entry is letter, runs left, @ the tick it may run again):")
			fmt.Printf("  %4s  %-10s  %-14s %s\n", "tick", "heap", "cooling", "ran")
		}
		fmt.Printf("  %4d  %-10s  %-14s %s\n", tick, heapString(h), queueString(q), ran)
	})
	fmt.Printf("schedule %s, length %d, closed form %d\n", s, len(s), closedForm(countsOf("AAABBBC"), 2))

	// 20,000 random cases: simulation == closed form, and the schedule is legal.
	var wrongLen, illegal, idleBound, totalTasks int
	for range 20000 {
		counts := make([]int, 1+r.IntN(26))
		tasks := 0
		for i := range counts {
			counts[i] = 1 + r.IntN(8)
			tasks += counts[i]
		}
		n := r.IntN(6)
		s, _ := schedule(counts, n, nil)
		if len(s) != closedForm(counts, n) {
			wrongLen++
		}
		if !validSchedule(s, counts, n) {
			illegal++
		}
		if len(s) > tasks {
			idleBound++
		}
		totalTasks += tasks
	}
	fmt.Println()
	fmt.Printf("20000 random cases (1..26 letters, counts 1..8, n 0..5; %d tasks in all)\n", totalTasks)
	fmt.Printf("heap simulation length != closed form: %d\n", wrongLen)
	fmt.Printf("schedules that break the cooldown or the counts: %d\n", illegal)
	fmt.Printf("cases that needed idle ticks: %d (the other %d ran with none)\n", idleBound, 20000-idleBound)

	// 300 tiny cases against every possible schedule.
	var tinyWrong, tinyNodes, tinyIdle int
	for range 300 {
		counts := make([]int, 1+r.IntN(3))
		for i := range counts {
			counts[i] = 1 + r.IntN(3)
		}
		n := r.IntN(4)
		s, _ := schedule(counts, n, nil)
		best, nodes := exhaustive(counts, n)
		tinyNodes += nodes
		if len(s) != best {
			tinyWrong++
		}
		if strings.Contains(s, "_") {
			tinyIdle++
		}
	}
	fmt.Println()
	fmt.Printf("300 tiny cases (1..3 letters, counts 1..3, n 0..3) against an exhaustive search of every schedule\n")
	fmt.Printf("length mismatches: %d (%d of the cases needed an idle tick; the search visited %d nodes)\n", tinyWrong, tinyIdle, tinyNodes)

	// The cost: one pop per task, a heap that never holds more than 26.
	fmt.Println()
	fmt.Println("1,000 tasks, 26 letters:")
	fmt.Printf("  %-26s %5s %6s %6s %6s %8s %9s\n", "shape", "n", "pushes", "pops", "max h", "length", "closed")
	for _, shape := range []string{"uniform", "A has 300"} {
		counts := make([]int, 26)
		n := 4
		if shape == "uniform" {
			for range 1000 {
				counts[r.IntN(26)]++
			}
		} else {
			counts[0] = 300
			for range 700 {
				counts[1+r.IntN(25)]++
			}
			n = 3
		}
		s, st := schedule(counts, n, nil)
		fmt.Printf("  %-26s %5d %6d %6d %6d %8d %9d\n", shape, n, st.pushes, st.pops, st.maxHeap, len(s), closedForm(counts, n))
	}
	fmt.Println("every task is popped exactly once, so the work is T pops on a heap of at most 26: Theta(T)")
}
```

**Output:**

```
== (a) meeting rooms: a min-heap of end times ==

meetings [start,end):  A[0,5) B[1,4) C[3,9) D[4,6) E[9,12) F[10,11)
  meeting   what happened                  heap of end times after (sorted)
  A[0,5)    new room                       [5]
  B[1,4)    new room                       [4 5]
  C[3,9)    new room                       [4 5 9]
  D[4,6)    reuse the room freed at 4      [5 6 9]
  E[9,12)   reuse the room freed at 5      [6 9 12]
  F[10,11)  reuse the room freed at 6      [9 11 12]
rooms needed: 3 (the final heap size)
D starts at 4 exactly when B ends, and takes B's room: [start,end) is half-open

2000 random cases (40556 meetings in all, start 0..29, length 1..8, so ties are common)
mismatches against the sorted-starts/sorted-ends sweep: 0
mismatches against "most meetings containing one instant": 0
busiest case needed 15 rooms; cases needing 0..15 rooms: 54 118 150 165 194 215 218 239 236 187 111 70 24 13 3 3
treating end == start as a clash (closed intervals) gives a different answer in 1165 of 2000 cases

== (b) task scheduler with a cooldown: a max-heap + a FIFO of cooling tasks ==

AAABBB, n=2 -> AB_AB_AB (length 8, closed form 8)
AAABBBC, n=2, tick by tick (heap and cooling shown after the tick; a cooling entry is letter, runs left, @ the tick it may run again):
  tick  heap        cooling        ran
     0  B3 C1       A2@3           A
     1  C1          A2@3 B2@4      B
     2              A2@3 B2@4      C
     3              B2@4 A1@6      A
     4              A1@6 B1@7      B
     5              A1@6 B1@7      _
     6              B1@7           A
     7                             B
schedule ABCAB_AB, length 8, closed form 8

20000 random cases (1..26 letters, counts 1..8, n 0..5; 1215228 tasks in all)
heap simulation length != closed form: 0
schedules that break the cooldown or the counts: 0
cases that needed idle ticks: 3330 (the other 16670 ran with none)

300 tiny cases (1..3 letters, counts 1..3, n 0..3) against an exhaustive search of every schedule
length mismatches: 0 (134 of the cases needed an idle tick; the search visited 20325 nodes)

1,000 tasks, 26 letters:
  shape                          n pushes   pops  max h   length    closed
  uniform                        4   1000   1000     26     1000      1000
  A has 300                      3   1000   1000     26     1197      1197
every task is popped exactly once, so the work is T pops on a heap of at most 26: Theta(T)
```

**Rust:**

```rust
use std::cmp::Reverse;
use std::collections::{BTreeMap, BinaryHeap, HashMap, VecDeque};

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so this program draws exactly the inputs the Go program draws.
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
        assert!(n > 0, "invalid argument to int_n");
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

// "Of the things I can do right now, which matters most?" is a priority queue.
// Two classic shapes, each proved against oracles instead of trusted:
//
//   (a) meeting rooms:  a min-heap of END times; its size is the rooms in use
//   (b) task cooldown:  a max-heap of remaining counts + a FIFO of cooling tasks

// ---------------------------------------------------------------- (a) rooms

type Meeting = (i32, i32); // half-open: [start, end)

fn ints(v: &[i32]) -> String {
    let parts: Vec<String> = v.iter().map(i32::to_string).collect();
    format!("[{}]", parts.join(" "))
}

/// rooms sorts by start, then keeps the end time of every room in use in a heap.
/// The root is the room that frees up first: if it is free by this meeting's
/// start, reuse it (pop, push the new end); otherwise open a room (push).
/// `touching` says whether end == start still counts as a clash (a closed
/// interval); the real rule for [start, end) is that it does not.
/// std's BinaryHeap is a max-heap, so `Reverse` turns it into the min-heap.
fn rooms(
    ms: &[Meeting],
    touching: bool,
    mut trace: impl FnMut(Meeting, &str, &BinaryHeap<Reverse<i32>>),
) -> usize {
    let mut sorted = ms.to_vec();
    sorted.sort(); // (start, end) lexicographic, the order the Go version sorts by
    let mut h: BinaryHeap<Reverse<i32>> = BinaryHeap::new();
    for m in sorted {
        let mut note = String::from("new room");
        if let Some(&Reverse(top)) = h.peek()
            && (top < m.0 || (!touching && top == m.0))
        {
            h.pop();
            note = format!("reuse the room freed at {top}");
        }
        h.push(Reverse(m.1));
        trace(m, &note, &h);
    }
    h.len() // it never shrinks below the number of rooms ever needed
}

// Oracle 1: sweep two sorted lists. Every start either reuses the earliest
// unmatched end (ends[j] <= start) or needs a new room.
fn rooms_sweep(ms: &[Meeting]) -> usize {
    let mut starts: Vec<i32> = ms.iter().map(|m| m.0).collect();
    let mut ends: Vec<i32> = ms.iter().map(|m| m.1).collect();
    starts.sort_unstable();
    ends.sort_unstable();
    let (mut need, mut j) = (0, 0);
    for s in starts {
        if ends[j] <= s {
            j += 1;
        } else {
            need += 1;
        }
    }
    need
}

// Oracle 2: the definition. Rooms needed = the most meetings that contain a
// single instant t (start <= t < end). Obviously right, O(n * horizon).
fn rooms_by_instant(ms: &[Meeting]) -> usize {
    (0..64)
        .map(|t| ms.iter().filter(|m| m.0 <= t && t < m.1).count())
        .max()
        .unwrap_or(0)
}

// ------------------------------------------------------------- (b) cooldown

// A task type: how many runs remain, and its letter. Max-heap on count; ties
// go to the smaller letter, so the order is strict and the schedule is unique.
// Rust gets the order from the tuple: (count, Reverse(letter)) is largest for
// the biggest count and, among equal counts, the smallest letter.
type Task = (u32, Reverse<u8>);
type TaskHeap = BinaryHeap<Task>;

// cooling is a task that ran and may run again at tick `ready`.
struct Cooling {
    ready: usize,
    count: u32,
    letter: u8,
}

#[derive(Default)]
struct Stats {
    pushes: usize,
    pops: usize,
    max_heap: usize,
}

/// schedule runs one tick at a time: first let tasks whose cooldown is over back
/// into the heap, then run the most urgent one, or idle. Ready times are
/// t+n+1 with t increasing, so a plain FIFO is already sorted by ready time.
fn schedule(
    counts: &[u32],
    n: usize,
    mut trace: impl FnMut(usize, &TaskHeap, &VecDeque<Cooling>, &str),
) -> (String, Stats) {
    let mut st = Stats::default();
    let mut h: TaskHeap = BinaryHeap::new();
    for (i, &c) in counts.iter().enumerate() {
        if c > 0 {
            h.push((c, Reverse(b'A' + i as u8)));
            st.pushes += 1;
        }
    }
    let mut q: VecDeque<Cooling> = VecDeque::new();
    let mut out = String::new();
    let mut tick = 0;
    while !h.is_empty() || !q.is_empty() {
        st.max_heap = st.max_heap.max(h.len());
        while let Some(front) = q.front()
            && front.ready <= tick
        {
            let c = q.pop_front().expect("front() was Some");
            h.push((c.count, Reverse(c.letter)));
            st.pushes += 1;
        }
        st.max_heap = st.max_heap.max(h.len());
        let mut ran = String::from("_");
        if let Some((count, Reverse(letter))) = h.pop() {
            st.pops += 1;
            ran = char::from(letter).to_string();
            if count > 1 {
                q.push_back(Cooling {
                    ready: tick + n + 1,
                    count: count - 1,
                    letter,
                });
            }
        }
        out.push_str(&ran);
        trace(tick, &h, &q, &ran);
        tick += 1;
    }
    (out, st)
}

/// closed_form: the busiest letter forces (maxCount-1) full frames of n+1 slots
/// and a last partial frame with one slot per letter tied for the maximum;
/// if there are more tasks than that, no idle slot is ever needed.
fn closed_form(counts: &[u32], n: usize) -> usize {
    let total: u32 = counts.iter().sum();
    let max_c = counts.iter().copied().max().unwrap_or(0);
    let num_max = counts.iter().filter(|&&c| c == max_c).count();
    if total == 0 {
        return 0;
    }
    (total as usize).max((max_c as usize - 1) * (n + 1) + num_max)
}

/// valid_schedule checks the three rules independently of the simulation: every
/// task type is run exactly its count, and two runs of one letter are at least
/// n+1 ticks apart.
fn valid_schedule(s: &str, counts: &[u32], n: usize) -> bool {
    let mut last: HashMap<u8, usize> = HashMap::new();
    let mut ran: HashMap<u8, u32> = HashMap::new();
    for (i, c) in s.bytes().enumerate() {
        if c == b'_' {
            continue;
        }
        if let Some(&p) = last.get(&c)
            && i - p < n + 1
        {
            return false;
        }
        last.insert(c, i);
        *ran.entry(c).or_default() += 1;
    }
    counts
        .iter()
        .enumerate()
        .all(|(i, &c)| ran.get(&(b'A' + i as u8)).copied().unwrap_or(0) == c)
}

// exhaustive tries EVERY schedule (any letter that is allowed, or idle, at
// every tick) and returns the shortest length. It prunes a branch only when it
// cannot beat the best so far: t ticks used + remaining tasks >= best.
// Go's version is a recursive closure; Rust cannot call a closure from inside
// itself, so the search state lives in a struct and `dfs` is a method.
struct Search {
    rem: Vec<u32>,
    next_ok: Vec<usize>,
    n: usize,
    best: usize,
    nodes: usize,
}

impl Search {
    fn dfs(&mut self, t: usize, left: usize) {
        self.nodes += 1;
        if left == 0 {
            self.best = self.best.min(t);
            return;
        }
        if t + left >= self.best {
            return;
        }
        for i in 0..self.rem.len() {
            if self.rem[i] > 0 && self.next_ok[i] <= t {
                let saved = self.next_ok[i];
                self.rem[i] -= 1;
                self.next_ok[i] = t + self.n + 1;
                self.dfs(t + 1, left - 1);
                self.rem[i] += 1;
                self.next_ok[i] = saved;
            }
        }
        self.dfs(t + 1, left); // idle, even when something could run
    }
}

fn exhaustive(counts: &[u32], n: usize) -> (usize, usize) {
    let left: usize = counts.iter().map(|&c| c as usize).sum();
    let mut s = Search {
        rem: counts.to_vec(),
        next_ok: vec![0; counts.len()],
        n,
        best: left * (n + 1) + 1, // a feasible length plus one: anything real beats it
        nodes: 0,
    };
    s.dfs(0, left);
    (s.best, s.nodes)
}

fn heap_string(h: &TaskHeap) -> String {
    let mut v: Vec<&Task> = h.iter().collect();
    v.sort_by(|a, b| b.cmp(a)); // most urgent first, the order they would pop in
    let parts: Vec<String> = v
        .iter()
        .map(|&&(c, Reverse(l))| format!("{}{}", char::from(l), c))
        .collect();
    parts.join(" ")
}

fn queue_string(q: &VecDeque<Cooling>) -> String {
    let parts: Vec<String> = q
        .iter()
        .map(|c| format!("{}{}@{}", char::from(c.letter), c.count, c.ready))
        .collect();
    parts.join(" ")
}

fn counts_of(word: &str) -> Vec<u32> {
    let mut counts = vec![0; 26];
    for b in word.bytes() {
        counts[usize::from(b - b'A')] += 1;
    }
    counts
}

fn main() {
    println!("== (a) meeting rooms: a min-heap of end times ==");
    println!();
    let ms: Vec<Meeting> = vec![(0, 5), (1, 4), (3, 9), (4, 6), (9, 12), (10, 11)];
    println!("meetings [start,end):  A[0,5) B[1,4) C[3,9) D[4,6) E[9,12) F[10,11)");
    println!(
        "  {:<9} {:<30} heap of end times after (sorted)",
        "meeting", "what happened"
    );
    let names = b"ABCDEF";
    let mut k = 0;
    let need = rooms(&ms, false, |m, note, h| {
        let label = format!("{}[{},{})", char::from(names[k]), m.0, m.1);
        let mut v: Vec<i32> = h.iter().map(|r| r.0).collect();
        v.sort_unstable();
        println!("  {:<9} {:<30} {}", label, note, ints(&v));
        k += 1;
    });
    println!("rooms needed: {need} (the final heap size)");
    println!("D starts at 4 exactly when B ends, and takes B's room: [start,end) is half-open");

    let mut r = Pcg::new(1, 2);
    const CASES: usize = 2000;
    let (mut mism1, mut mism2, mut closed_differs, mut total, mut widest) = (0, 0, 0, 0, 0);
    let mut hist: BTreeMap<usize, usize> = BTreeMap::new();
    for _ in 0..CASES {
        let n = r.int_n(41);
        let mut ms: Vec<Meeting> = Vec::with_capacity(n);
        for _ in 0..n {
            let s = r.int_n(30) as i32;
            ms.push((s, s + 1 + r.int_n(8) as i32));
        }
        let got = rooms(&ms, false, |_, _, _| {});
        if got != rooms_sweep(&ms) {
            mism1 += 1;
        }
        if got != rooms_by_instant(&ms) {
            mism2 += 1;
        }
        if rooms(&ms, true, |_, _, _| {}) != got {
            closed_differs += 1;
        }
        total += ms.len();
        widest = widest.max(got);
        *hist.entry(got).or_default() += 1;
    }
    println!();
    println!(
        "{CASES} random cases ({total} meetings in all, start 0..29, length 1..8, so ties are common)"
    );
    println!("mismatches against the sorted-starts/sorted-ends sweep: {mism1}");
    println!("mismatches against \"most meetings containing one instant\": {mism2}");
    let counts: Vec<String> = (0..=widest)
        .map(|w| hist.get(&w).copied().unwrap_or(0).to_string())
        .collect();
    println!(
        "busiest case needed {widest} rooms; cases needing 0..{widest} rooms: {}",
        counts.join(" ")
    );
    println!(
        "treating end == start as a clash (closed intervals) gives a different answer in {closed_differs} of {CASES} cases"
    );

    println!();
    println!("== (b) task scheduler with a cooldown: a max-heap + a FIFO of cooling tasks ==");
    println!();
    let (s, _) = schedule(&counts_of("AAABBB"), 2, |_, _, _, _| {});
    println!(
        "AAABBB, n=2 -> {s} (length {}, closed form {})",
        s.len(),
        closed_form(&counts_of("AAABBB"), 2)
    );
    let (s, _) = schedule(&counts_of("AAABBBC"), 2, |tick, h, q, ran| {
        if tick == 0 {
            println!(
                "AAABBBC, n=2, tick by tick (heap and cooling shown after the tick; a cooling entry is letter, runs left, @ the tick it may run again):"
            );
            println!("  {:>4}  {:<10}  {:<14} ran", "tick", "heap", "cooling");
        }
        println!(
            "  {:>4}  {:<10}  {:<14} {}",
            tick,
            heap_string(h),
            queue_string(q),
            ran
        );
    });
    println!(
        "schedule {s}, length {}, closed form {}",
        s.len(),
        closed_form(&counts_of("AAABBBC"), 2)
    );

    // 20,000 random cases: simulation == closed form, and the schedule is legal.
    let (mut wrong_len, mut illegal, mut idle_bound, mut total_tasks) = (0, 0, 0, 0);
    for _ in 0..20000 {
        let letters = 1 + r.int_n(26);
        let mut counts = vec![0u32; letters];
        let mut tasks = 0;
        for c in counts.iter_mut() {
            *c = 1 + r.int_n(8) as u32;
            tasks += *c as usize;
        }
        let n = r.int_n(6);
        let (s, _) = schedule(&counts, n, |_, _, _, _| {});
        if s.len() != closed_form(&counts, n) {
            wrong_len += 1;
        }
        if !valid_schedule(&s, &counts, n) {
            illegal += 1;
        }
        if s.len() > tasks {
            idle_bound += 1;
        }
        total_tasks += tasks;
    }
    println!();
    println!("20000 random cases (1..26 letters, counts 1..8, n 0..5; {total_tasks} tasks in all)");
    println!("heap simulation length != closed form: {wrong_len}");
    println!("schedules that break the cooldown or the counts: {illegal}");
    println!(
        "cases that needed idle ticks: {idle_bound} (the other {} ran with none)",
        20000 - idle_bound
    );

    // 300 tiny cases against every possible schedule.
    let (mut tiny_wrong, mut tiny_nodes, mut tiny_idle) = (0, 0, 0);
    for _ in 0..300 {
        let letters = 1 + r.int_n(3);
        let mut counts = vec![0u32; letters];
        for c in counts.iter_mut() {
            *c = 1 + r.int_n(3) as u32;
        }
        let n = r.int_n(4);
        let (s, _) = schedule(&counts, n, |_, _, _, _| {});
        let (best, nodes) = exhaustive(&counts, n);
        tiny_nodes += nodes;
        if s.len() != best {
            tiny_wrong += 1;
        }
        if s.contains('_') {
            tiny_idle += 1;
        }
    }
    println!();
    println!(
        "300 tiny cases (1..3 letters, counts 1..3, n 0..3) against an exhaustive search of every schedule"
    );
    println!(
        "length mismatches: {tiny_wrong} ({tiny_idle} of the cases needed an idle tick; the search visited {tiny_nodes} nodes)"
    );

    // The cost: one pop per task, a heap that never holds more than 26.
    println!();
    println!("1,000 tasks, 26 letters:");
    println!(
        "  {:<26} {:>5} {:>6} {:>6} {:>6} {:>8} {:>9}",
        "shape", "n", "pushes", "pops", "max h", "length", "closed"
    );
    for shape in ["uniform", "A has 300"] {
        let mut counts = vec![0u32; 26];
        let mut n = 4;
        if shape == "uniform" {
            for _ in 0..1000 {
                counts[r.int_n(26)] += 1;
            }
        } else {
            counts[0] = 300;
            for _ in 0..700 {
                counts[1 + r.int_n(25)] += 1;
            }
            n = 3;
        }
        let (s, st) = schedule(&counts, n, |_, _, _, _| {});
        println!(
            "  {:<26} {:>5} {:>6} {:>6} {:>6} {:>8} {:>9}",
            shape,
            n,
            st.pushes,
            st.pops,
            st.max_heap,
            s.len(),
            closed_form(&counts, n)
        );
    }
    println!(
        "every task is popped exactly once, so the work is T pops on a heap of at most 26: Theta(T)"
    );
}
```

**Rust output:**

```
== (a) meeting rooms: a min-heap of end times ==

meetings [start,end):  A[0,5) B[1,4) C[3,9) D[4,6) E[9,12) F[10,11)
  meeting   what happened                  heap of end times after (sorted)
  A[0,5)    new room                       [5]
  B[1,4)    new room                       [4 5]
  C[3,9)    new room                       [4 5 9]
  D[4,6)    reuse the room freed at 4      [5 6 9]
  E[9,12)   reuse the room freed at 5      [6 9 12]
  F[10,11)  reuse the room freed at 6      [9 11 12]
rooms needed: 3 (the final heap size)
D starts at 4 exactly when B ends, and takes B's room: [start,end) is half-open

2000 random cases (40556 meetings in all, start 0..29, length 1..8, so ties are common)
mismatches against the sorted-starts/sorted-ends sweep: 0
mismatches against "most meetings containing one instant": 0
busiest case needed 15 rooms; cases needing 0..15 rooms: 54 118 150 165 194 215 218 239 236 187 111 70 24 13 3 3
treating end == start as a clash (closed intervals) gives a different answer in 1165 of 2000 cases

== (b) task scheduler with a cooldown: a max-heap + a FIFO of cooling tasks ==

AAABBB, n=2 -> AB_AB_AB (length 8, closed form 8)
AAABBBC, n=2, tick by tick (heap and cooling shown after the tick; a cooling entry is letter, runs left, @ the tick it may run again):
  tick  heap        cooling        ran
     0  B3 C1       A2@3           A
     1  C1          A2@3 B2@4      B
     2              A2@3 B2@4      C
     3              B2@4 A1@6      A
     4              A1@6 B1@7      B
     5              A1@6 B1@7      _
     6              B1@7           A
     7                             B
schedule ABCAB_AB, length 8, closed form 8

20000 random cases (1..26 letters, counts 1..8, n 0..5; 1215228 tasks in all)
heap simulation length != closed form: 0
schedules that break the cooldown or the counts: 0
cases that needed idle ticks: 3330 (the other 16670 ran with none)

300 tiny cases (1..3 letters, counts 1..3, n 0..3) against an exhaustive search of every schedule
length mismatches: 0 (134 of the cases needed an idle tick; the search visited 20325 nodes)

1,000 tasks, 26 letters:
  shape                          n pushes   pops  max h   length    closed
  uniform                        4   1000   1000     26     1000      1000
  A has 300                      3   1000   1000     26     1197      1197
every task is popped exactly once, so the work is T pops on a heap of at most 26: Theta(T)
```

> **Go vs Rust:** The output is identical to Go's, line for line (same PCG stream, same draws). `BinaryHeap` is a max-heap, so end times go in as `Reverse<i32>` and the task heap is a `BinaryHeap<(u32, Reverse<u8>)>`: tuple order alone gives "biggest count, then smallest letter", where Go needs a `Less` method plus `Len`/`Swap`/`Push`/`Pop` and an `any` round trip (`heap.Pop(h).(task)`); Rust's `pop` returns the typed `Option`. The heap contents are printed sorted in both programs, because the two libraries lay out the same set differently in the array. Go's recursive closure for the exhaustive search becomes a struct with a `dfs` method, since a Rust closure cannot call itself.

**Complexity:** rooms Θ(n log n), the sort, with a heap of at most n end times (the final heap size is the answer: 3 rooms for the worked case, at most 15 in the 2,000 random ones) · cooldown Θ(T log 26) = Θ(T): 1,000 pops for 1,000 tasks, never more than 26 letters in the heap · 0 mismatches against every oracle, the closed form included, and the half-open rule matters: counting `end == start` as a clash changes the answer in 1,165 of the 2,000 cases.

---

## 11. Cancel and reschedule: lazy deletion and what it leaks

`🟡 medium` · *No decrease-key*

Neither standard library has decrease-key. Go gives `heap.Fix(h, i)` and `heap.Remove(h, i)` by **index**, so every element must know where it sits (you keep `index` up to date inside `Swap`); Rust's `BinaryHeap` gives nothing (`peek_mut` reaches only the top, `retain` and `into_vec` + `from` are O(n)). Change a key in place without telling the heap and you get a wrong pop order and no error. The portable answer is **lazy deletion**: mark the entry dead, leave it inside, discard it when it surfaces. Its bill comes in two currencies. A stale entry that surfaces costs one extra pop; one that never surfaces costs space. A timer queue shows both (50,000 ticks, nine in ten events cancelled soon after they are scheduled), next to `Remove` by index and next to lazy deletion with **compaction** (drop the dead entries and re-heapify when the heap holds more than 2 × live + 16). `Remove` and `Fix` are Θ(log n); a lazy cancel is Θ(1) and its price is paid later.

**Steps:**

1. Build a six-item heap whose items know their index, change one priority in place and watch the array and the pop order go wrong, then repair it with `Fix`, and take another item out with `Remove`.
2. Replay one seeded timer workload three ways (`Remove` by index, lazy, lazy + compact), twice: once with events due 1,000 ticks out (stale entries surface) and once with events due 100,000 ticks out (nothing ever fires). Count max heap size, extra pops, entries left behind, rebuilds and `Less` calls by operation, and check every strategy against an oracle.
3. Read the bill: how much bigger the lazy heap gets, how many `Less` calls the stale pops cost when they surface, and what the same entries cost (no pops, about 90% of the heap) when they never do.

```go
package main

import (
	"container/heap"
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
)

// No standard library has decrease-key. Go gives Fix(h, i) and Remove(h, i)
// BY INDEX, so the element must know where it sits (kept up to date in Swap);
// Rust gives nothing at all. The portable answer is lazy deletion: mark the
// entry dead, leave it in the heap, throw it away when it surfaces. This
// program counts what that costs: the extra pops, and the dead entries that
// never surface.

// Event is a timer due at tick `at`. The heap order is the strict total order
// (at, id), so the pop sequence does not depend on how the heap is laid out.
type Event struct {
	at, id int
	index  int  // slot in the heap array, -1 when it is not in the heap
	dead   bool // cancelled, but still stored (lazy deletion)
}

func before(x, y *Event) bool {
	if x.at != y.at {
		return x.at < y.at
	}
	return x.id < y.id
}

// EventHeap is container/heap's Interface, with the index kept in Swap/Push/Pop
// the way the package's own example does, and a counter on Less.
type EventHeap struct {
	a    []*Event
	less int
}

func (h *EventHeap) Len() int { return len(h.a) }
func (h *EventHeap) Less(i, j int) bool {
	h.less++
	return before(h.a[i], h.a[j])
}
func (h *EventHeap) Swap(i, j int) {
	h.a[i], h.a[j] = h.a[j], h.a[i]
	h.a[i].index = i
	h.a[j].index = j
}
func (h *EventHeap) Push(x any) {
	e := x.(*Event)
	e.index = len(h.a)
	h.a = append(h.a, e)
}
func (h *EventHeap) Pop() any {
	n := len(h.a)
	e := h.a[n-1]
	h.a[n-1] = nil // do not keep the event alive through the backing array
	e.index = -1
	h.a = h.a[:n-1]
	return e
}

// compact drops every dead entry and re-heapifies the survivors (Floyd, Theta(n)).
func (h *EventHeap) compact() int {
	kept := h.a[:0]
	for _, e := range h.a {
		if e.dead {
			e.index = -1
			continue
		}
		e.index = len(kept)
		kept = append(kept, e)
	}
	removed := len(h.a) - len(kept)
	clear(h.a[len(kept):])
	h.a = kept
	heap.Init(h)
	return removed
}

// valid: the heap property holds and every event's index is its real slot.
func (h *EventHeap) valid() bool {
	for i, e := range h.a {
		if e.index != i || (i > 0 && before(e, h.a[(i-1)/2])) {
			return false
		}
	}
	return true
}

// ------------------------------------------------ (a) change a key in place

func partA() {
	names := "abcdef"
	build := func() (*EventHeap, []*Event) {
		h := &EventHeap{}
		items := make([]*Event, 6)
		for i := range items {
			items[i] = &Event{at: i + 1, id: i} // `at` plays the priority here
			heap.Push(h, items[i])
		}
		return h, items
	}
	show := func(label string, h *EventHeap) {
		var layout, order []string
		for _, e := range h.a {
			layout = append(layout, fmt.Sprintf("%c%d", names[e.id], e.at))
		}
		ok := h.valid()
		sorted := true
		prev := -1 << 31
		for h.Len() > 0 {
			e := heap.Pop(h).(*Event)
			order = append(order, fmt.Sprintf("%c(%d)", names[e.id], e.at))
			sorted = sorted && e.at >= prev
			prev = e.at
		}
		verdict := ""
		if !sorted {
			verdict = "   <- not in priority order"
		}
		fmt.Printf("  %-22s [%s]  heap=%-5v pops: %s%s\n",
			label, strings.Join(layout, " "), ok, strings.Join(order, " "), verdict)
	}

	h, _ := build()
	show("start: a1 ... f6", h)

	h, items := build()
	items[3].at = 0 // d is now the most urgent, and nobody told the heap
	show("d.at = 0, no Fix", h)

	h, items = build()
	items[3].at = 0
	heap.Fix(h, items[3].index)
	show("d.at = 0, then Fix", h)

	h, items = build()
	removed := heap.Remove(h, items[2].index).(*Event)
	fmt.Printf("  heap.Remove(h, c.index) returned %c(%d); its index is now %d; the rest is still a heap: %v\n",
		names[removed.id], removed.at, removed.index, h.valid())
	show("after Remove(c)", h)
}

// ------------------------------------------ (b) the timer workload, 3 ways

const ticks = 50000

// workload: every tick schedules one event; with probability 9/10 it is
// cancelled 1..10 ticks later. cancelAt[t] lists the ids cancelled at tick t.
type workload struct {
	cancelAt  [][]int
	cancelled []bool // the cancel is delivered before the run ends
}

func newWorkload(r *rand.Rand) *workload {
	w := &workload{cancelAt: make([][]int, ticks+11), cancelled: make([]bool, ticks)}
	for id := range ticks {
		if r.IntN(10) < 9 {
			t := id + 1 + r.IntN(10)
			w.cancelAt[t] = append(w.cancelAt[t], id)
			w.cancelled[id] = t < ticks
		}
	}
	return w
}

type strategy int

const (
	fix strategy = iota
	lazy
	lazyCompact
)

var strategyNames = []string{"fix (Remove)", "lazy", "lazy + compact"}

type result struct {
	maxHeap, fired, cancelled, stalePops, compacted, staleLeft, rebuilds int
	lessPush, lessPop, lessCancel, lessRebuild                           int
	firedIDs                                                             []int
	valid                                                                bool
}

func (r *result) less() int { return r.lessPush + r.lessPop + r.lessCancel + r.lessRebuild }

// run replays the workload: each tick schedules an event due `horizon` ticks
// later, delivers the cancels due now, then fires everything due (at <= now).
// Cancels arrive at most 10 ticks after the schedule and every event is due
// >= 1000 ticks out, so a cancel never meets an event that already fired.
func run(s strategy, horizon int, w *workload) result {
	var res result
	h := &EventHeap{}
	events := make([]*Event, ticks)
	live := 0
	spent := func(counter *int, f func()) { // Less calls made by f
		before := h.less
		f()
		*counter += h.less - before
	}
	for now := range ticks {
		e := &Event{at: now + horizon, id: now}
		events[now] = e
		spent(&res.lessPush, func() { heap.Push(h, e) })
		live++
		res.maxHeap = max(res.maxHeap, h.Len())

		for _, id := range w.cancelAt[now] {
			res.cancelled++
			live--
			if s == fix {
				spent(&res.lessCancel, func() { heap.Remove(h, events[id].index) })
				continue
			}
			events[id].dead = true // lazy: it stays in the heap
			if s == lazyCompact && h.Len() > 2*live+16 {
				spent(&res.lessRebuild, func() { res.compacted += h.compact() })
				res.rebuilds++
			}
		}

		for h.Len() > 0 && h.a[0].at <= now {
			var top *Event
			spent(&res.lessPop, func() { top = heap.Pop(h).(*Event) })
			if top.dead {
				res.stalePops++
				continue
			}
			res.fired++
			live--
			res.firedIDs = append(res.firedIDs, top.id)
		}
	}
	for _, e := range h.a {
		if e.dead {
			res.staleLeft++
		}
	}
	res.valid = h.valid()
	return res
}

func main() {
	fmt.Println("== (a) changing a priority while the item is in the heap ==")
	fmt.Println()
	partA()

	fmt.Println()
	fmt.Println("== (b) timers: 50,000 ticks, 9 in 10 events cancelled 1..10 ticks after they are scheduled ==")
	r := rand.New(rand.NewPCG(1, 2))
	w := newWorkload(r)
	delivered := 0
	for _, c := range w.cancelled {
		if c {
			delivered++
		}
	}
	fmt.Printf("%d events scheduled, %d cancels delivered before the run ends\n", ticks, delivered)

	for _, horizon := range []int{1000, 100000} {
		fmt.Println()
		fmt.Printf("every event due %d ticks after it is scheduled", horizon)
		if horizon > ticks {
			fmt.Printf(" (longer than the run: nothing ever fires)")
		}
		fmt.Println()

		// The oracle: the events not cancelled whose due tick is inside the run,
		// in (at, id) order.
		var want []int
		for id := range ticks {
			if !w.cancelled[id] && id+horizon < ticks {
				want = append(want, id)
			}
		}
		var res [3]result
		for s := range res {
			res[s] = run(strategy(s), horizon, w)
		}
		fmt.Printf("  %-15s %8s %7s %9s %10s %10s %11s\n",
			"strategy", "max heap", "fired", "cancelled", "stale pops", "compacted", "stale left")
		for s, x := range res {
			fmt.Printf("  %-15s %8d %7d %9d %10d %10d %11d\n",
				strategyNames[s], x.maxHeap, x.fired, x.cancelled, x.stalePops, x.compacted, x.staleLeft)
		}
		fmt.Printf("  %-15s %8s %10s %9s %9s %9s %11s\n",
			"strategy", "rebuilds", "Less total", "in Push", "in Pop", "in Remove", "in rebuilds")
		for s, x := range res {
			fmt.Printf("  %-15s %8d %10d %9d %9d %9d %11d\n",
				strategyNames[s], x.rebuilds, x.less(), x.lessPush, x.lessPop, x.lessCancel, x.lessRebuild)
		}
		same, valid := true, true
		for _, x := range res {
			same = same && slices.Equal(x.firedIDs, want)
			valid = valid && x.valid
		}
		// fix removes each cancelled event outright; every lazy cancel must be accounted for.
		conserved := true
		for _, x := range res[lazy:] {
			conserved = conserved && x.cancelled == x.stalePops+x.compacted+x.staleLeft
		}
		fmt.Printf("  every strategy fired exactly the %d events the oracle expects, in order: %v\n", len(want), same)
		fmt.Printf("  lazy rows: cancelled == stale pops + compacted + stale left: %v; heaps valid at the end: %v\n", conserved, valid)
		fmt.Printf("  lazy peaked at %.1fx the heap that Remove needed (%d vs %d entries)",
			float64(res[lazy].maxHeap)/float64(res[fix].maxHeap), res[lazy].maxHeap, res[fix].maxHeap)
		fmt.Printf("; compaction at %.1fx (%d)\n", float64(res[lazyCompact].maxHeap)/float64(res[fix].maxHeap), res[lazyCompact].maxHeap)
		fmt.Printf("  Less calls against fix: lazy %.1fx, lazy + compact %.1fx\n",
			float64(res[lazy].less())/float64(res[fix].less()), float64(res[lazyCompact].less())/float64(res[fix].less()))
	}
}
```

**Output:**

```
== (a) changing a priority while the item is in the heap ==

  start: a1 ... f6       [a1 b2 c3 d4 e5 f6]  heap=true  pops: a(1) b(2) c(3) d(4) e(5) f(6)
  d.at = 0, no Fix       [a1 b2 c3 d0 e5 f6]  heap=false pops: a(1) b(2) d(0) c(3) e(5) f(6)   <- not in priority order
  d.at = 0, then Fix     [d0 a1 c3 b2 e5 f6]  heap=true  pops: d(0) a(1) b(2) c(3) e(5) f(6)
  heap.Remove(h, c.index) returned c(3); its index is now -1; the rest is still a heap: true
  after Remove(c)        [a1 b2 f6 d4 e5]  heap=true  pops: a(1) b(2) d(4) e(5) f(6)

== (b) timers: 50,000 ticks, 9 in 10 events cancelled 1..10 ticks after they are scheduled ==
50000 events scheduled, 44964 cancels delivered before the run ends

every event due 1000 ticks after it is scheduled
  strategy        max heap   fired cancelled stale pops  compacted  stale left
  fix (Remove)         135    4916     44964          0          0           0
  lazy                1001    4916     44964      44084          0         880
  lazy + compact       276    4916     44964          0      44828         136
  strategy        rebuilds Less total   in Push    in Pop in Remove in rebuilds
  fix (Remove)           0     150089     49999     56745     43345           0
  lazy                   0     929776     49999    879777         0           0
  lazy + compact       377     150487     49999     62139         0       38349
  every strategy fired exactly the 4916 events the oracle expects, in order: true
  lazy rows: cancelled == stale pops + compacted + stale left: true; heaps valid at the end: true
  lazy peaked at 7.4x the heap that Remove needed (1001 vs 135 entries); compaction at 2.0x (276)
  Less calls against fix: lazy 6.2x, lazy + compact 1.0x

every event due 100000 ticks after it is scheduled (longer than the run: nothing ever fires)
  strategy        max heap   fired cancelled stale pops  compacted  stale left
  fix (Remove)        5036       0     44964          0          0           0
  lazy               50000       0     44964          0          0       44964
  lazy + compact      9917       0     44964          0      44293         671
  strategy        rebuilds Less total   in Push    in Pop in Remove in rebuilds
  fix (Remove)           0      92542     49999         0     42543           0
  lazy                   0      49999     49999         0         0           0
  lazy + compact        46      93458     49999         0         0       43459
  every strategy fired exactly the 0 events the oracle expects, in order: true
  lazy rows: cancelled == stale pops + compacted + stale left: true; heaps valid at the end: true
  lazy peaked at 9.9x the heap that Remove needed (50000 vs 5036 entries); compaction at 2.0x (9917)
  Less calls against fix: lazy 0.5x, lazy + compact 1.0x
```

**Rust:**

```rust
use std::cell::Cell;
use std::cmp::{Ordering, Reverse};
use std::collections::BinaryHeap;

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so this program draws exactly the workload the Go program draws.
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
        assert!(n > 0, "invalid argument to int_n");
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

// No standard library has decrease-key. Go gives Fix(h, i) and Remove(h, i) BY
// INDEX; Rust's BinaryHeap gives nothing at all: peek_mut reaches only the top,
// retain and into_vec + from are O(n). The portable answer is lazy deletion:
// mark the entry dead, leave it in the heap, throw it away when it surfaces.
// This program counts what that costs: the extra pops, and the dead entries
// that never surface.

// ------------------------------------------------ (a) change a key in place

// An item whose key lives in a Cell. A BinaryHeap<&Item> holds only shared
// references, and a Cell can still be written through one: that is the only
// way safe Rust can change an element's order while the heap stores it. The
// docs call it a logic error: unspecified results, but no undefined behaviour.
struct Item {
    name: char,
    key: Cell<i32>,
}

// The smaller key is "greater", so the max-heap acts as Go's min-heap.
impl Ord for Item {
    fn cmp(&self, other: &Self) -> Ordering {
        (other.key.get(), other.name).cmp(&(self.key.get(), self.name))
    }
}

impl PartialOrd for Item {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl PartialEq for Item {
    fn eq(&self, other: &Self) -> bool {
        self.cmp(other) == Ordering::Equal
    }
}

impl Eq for Item {}

fn make_items() -> Vec<Item> {
    "abcdef"
        .chars()
        .zip(1..)
        .map(|(name, k)| Item {
            name,
            key: Cell::new(k),
        })
        .collect()
}

fn build(items: &[Item]) -> BinaryHeap<&Item> {
    let mut h = BinaryHeap::new();
    for it in items {
        h.push(it);
    }
    h
}

// show prints the array, whether the max-heap property holds, and the pop order.
fn show(label: &str, mut h: BinaryHeap<&Item>) {
    let s = h.as_slice();
    let layout: Vec<String> = s
        .iter()
        .map(|it| format!("{}{}", it.name, it.key.get()))
        .collect();
    let ok = (1..s.len()).all(|i| s[i] <= s[(i - 1) / 2]);
    let (mut order, mut sorted, mut prev) = (Vec::new(), true, i32::MIN);
    while let Some(it) = h.pop() {
        order.push(format!("{}({})", it.name, it.key.get()));
        sorted &= it.key.get() >= prev;
        prev = it.key.get();
    }
    let verdict = if sorted {
        ""
    } else {
        "   <- not in priority order"
    };
    println!(
        "  {:<22} [{}]  heap={:<5} pops: {}{}",
        label,
        layout.join(" "),
        ok,
        order.join(" "),
        verdict
    );
}

fn part_a() {
    let items = make_items();
    show("start: a1 ... f6", build(&items));

    let items = make_items();
    let h = build(&items);
    items[3].key.set(0); // d is now the most urgent, and nobody told the heap
    show("d.key = 0, no repair", h);

    // Rust has no Fix: the repair is a full O(n) rebuild of the array.
    let items = make_items();
    let h = build(&items);
    items[3].key.set(0);
    show("d.key = 0, rebuilt", BinaryHeap::from(h.into_vec()));

    // Rust has no Remove(i) either: retain filters the whole heap, O(n).
    let items = make_items();
    let mut h = build(&items);
    h.retain(|it| it.name != 'c');
    show("retain(name != c)", h);

    // The other guard-based way to break a heap: write through peek_mut, then
    // forget the guard so its Drop (the re-sift) never runs. Safe code can do it.
    let mut h = BinaryHeap::from([5, 4, 3, 2, 1]);
    let before = h.len();
    let mut top = h.peek_mut().expect("heap is not empty");
    *top = 0; // DerefMut: the heap's length is cut to 1 while the guard lives
    std::mem::forget(top);
    println!(
        "  forgetting a written-through PeekMut: len {} -> {}; pops: {:?} {:?} (the other four were leaked)",
        before,
        h.len(),
        h.pop(),
        h.pop()
    );
}

// ------------------------------------------ (b) the timer workload, 3 ways

const TICKS: usize = 50000;

// workload: every tick schedules one event; with probability 9/10 it is
// cancelled 1..10 ticks later. cancel_at[t] lists the ids cancelled at tick t.
struct Workload {
    cancel_at: Vec<Vec<usize>>,
    cancelled: Vec<bool>, // the cancel is delivered before the run ends
}

fn new_workload(r: &mut Pcg) -> Workload {
    let mut w = Workload {
        cancel_at: vec![Vec::new(); TICKS + 11],
        cancelled: vec![false; TICKS],
    };
    for id in 0..TICKS {
        if r.int_n(10) < 9 {
            let t = id + 1 + r.int_n(10);
            w.cancel_at[t].push(id);
            w.cancelled[id] = t < TICKS;
        }
    }
    w
}

#[derive(Clone, Copy, PartialEq)]
enum Mode {
    Fix,
    Lazy,
    LazyCompact,
}

// Timers is what the run loop needs from a priority queue of (at, id) events.
trait Timers {
    fn push(&mut self, at: usize, id: usize);
    fn peek_at(&self) -> Option<usize>;
    fn pop(&mut self) -> Option<usize>;
    /// remove by id: only a heap that knows where each event sits can do this.
    fn remove(&mut self, id: usize) -> bool;
    /// drop every dead entry and re-heapify; returns how many were dropped.
    fn compact(&mut self, dead: &[bool]) -> usize;
    fn len(&self) -> usize;
    fn dead_inside(&self, dead: &[bool]) -> usize;
    fn valid(&self) -> bool;
    fn less_calls(&self) -> u64;
}

// Heap is container/heap's algorithm, ported line for line from Go (up, down,
// Push, Pop, Remove, Init), with the position of every id kept in `pos`
// (Go's Event.index). Same code, same strict (at, id) order: the Less counts
// match Go's exactly.
struct Heap {
    data: Vec<(usize, usize)>, // (at, id)
    pos: Vec<Option<usize>>,   // pos[id] = slot in `data`, None when not stored
    less_calls: u64,
}

impl Heap {
    fn new() -> Self {
        Heap {
            data: Vec::new(),
            pos: vec![None; TICKS],
            less_calls: 0,
        }
    }

    fn less(&mut self, i: usize, j: usize) -> bool {
        self.less_calls += 1;
        self.data[i] < self.data[j]
    }

    fn swap(&mut self, i: usize, j: usize) {
        self.data.swap(i, j);
        self.pos[self.data[i].1] = Some(i);
        self.pos[self.data[j].1] = Some(j);
    }

    fn up(&mut self, mut j: usize) {
        while j > 0 {
            let i = (j - 1) / 2; // parent
            if !self.less(j, i) {
                break;
            }
            self.swap(i, j);
            j = i;
        }
    }

    fn down(&mut self, i0: usize, n: usize) -> bool {
        let mut i = i0;
        loop {
            let j1 = 2 * i + 1;
            if j1 >= n {
                break;
            }
            let mut j = j1; // left child
            let j2 = j1 + 1;
            if j2 < n && self.less(j2, j1) {
                j = j2; // right child, only if strictly less
            }
            if !self.less(j, i) {
                break;
            }
            self.swap(i, j);
            i = j;
        }
        i > i0
    }

    fn init(&mut self) {
        let n = self.data.len();
        for i in (0..n / 2).rev() {
            self.down(i, n);
        }
    }

    // the last slot leaves the array (Go's Pop method)
    fn take_last(&mut self) -> Option<usize> {
        let (_, id) = self.data.pop()?;
        self.pos[id] = None;
        Some(id)
    }
}

impl Timers for Heap {
    fn push(&mut self, at: usize, id: usize) {
        self.pos[id] = Some(self.data.len());
        self.data.push((at, id));
        self.up(self.data.len() - 1);
    }

    fn peek_at(&self) -> Option<usize> {
        self.data.first().map(|e| e.0)
    }

    fn pop(&mut self) -> Option<usize> {
        let n = self.data.len().checked_sub(1)?;
        self.swap(0, n);
        self.down(0, n);
        self.take_last()
    }

    fn remove(&mut self, id: usize) -> bool {
        let Some(i) = self.pos[id] else {
            return false;
        };
        let n = self.data.len() - 1;
        if n != i {
            self.swap(i, n);
            if !self.down(i, n) {
                self.up(i);
            }
        }
        self.take_last();
        true
    }

    fn compact(&mut self, dead: &[bool]) -> usize {
        let before = self.data.len();
        for &(_, id) in &self.data {
            if dead[id] {
                self.pos[id] = None;
            }
        }
        self.data.retain(|&(_, id)| !dead[id]); // keeps the survivors' relative order
        for (slot, &(_, id)) in self.data.iter().enumerate() {
            self.pos[id] = Some(slot);
        }
        self.init();
        before - self.data.len()
    }

    fn len(&self) -> usize {
        self.data.len()
    }

    fn dead_inside(&self, dead: &[bool]) -> usize {
        self.data.iter().filter(|e| dead[e.1]).count()
    }

    fn valid(&self) -> bool {
        self.data
            .iter()
            .enumerate()
            .all(|(i, e)| self.pos[e.1] == Some(i) && (i == 0 || self.data[(i - 1) / 2] <= *e))
    }

    fn less_calls(&self) -> u64 {
        self.less_calls
    }
}

// The `std` line: BinaryHeap<Reverse<Key>>. Key counts every comparison the
// library makes in a thread-local (std's <, <=, >= each call partial_cmp once,
// and partial_cmp delegates to cmp), so Less counts are comparable to Heap's.
thread_local!(static CMP: Cell<u64> = const { Cell::new(0) });

#[derive(PartialEq, Eq)]
struct Key(usize, usize); // (at, id)

impl Ord for Key {
    fn cmp(&self, other: &Self) -> Ordering {
        CMP.set(CMP.get() + 1);
        (self.0, self.1).cmp(&(other.0, other.1))
    }
}

impl PartialOrd for Key {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

struct StdTimers(BinaryHeap<Reverse<Key>>);

impl Timers for StdTimers {
    fn push(&mut self, at: usize, id: usize) {
        self.0.push(Reverse(Key(at, id)));
    }

    fn peek_at(&self) -> Option<usize> {
        self.0.peek().map(|r| r.0.0)
    }

    fn pop(&mut self) -> Option<usize> {
        self.0.pop().map(|r| r.0.1)
    }

    // std has nothing to call here, so the Fix strategy never runs on StdTimers.
    fn remove(&mut self, _id: usize) -> bool {
        false
    }

    // BinaryHeap::retain: stable since 1.70, O(n), rebuilds from the first removal.
    fn compact(&mut self, dead: &[bool]) -> usize {
        let before = self.0.len();
        self.0.retain(|r| !dead[r.0.1]);
        before - self.0.len()
    }

    fn len(&self) -> usize {
        self.0.len()
    }

    fn dead_inside(&self, dead: &[bool]) -> usize {
        self.0.iter().filter(|r| dead[r.0.1]).count()
    }

    fn valid(&self) -> bool {
        let s = self.0.as_slice();
        (1..s.len()).all(|i| s[i] <= s[(i - 1) / 2])
    }

    fn less_calls(&self) -> u64 {
        CMP.get()
    }
}

#[derive(Default)]
struct Res {
    max_heap: usize,
    fired: usize,
    cancelled: usize,
    stale_pops: usize,
    compacted: usize,
    stale_left: usize,
    rebuilds: usize,
    less_push: u64,
    less_pop: u64,
    less_cancel: u64,
    less_rebuild: u64,
    fired_ids: Vec<usize>,
    valid: bool,
}

impl Res {
    fn less(&self) -> u64 {
        self.less_push + self.less_pop + self.less_cancel + self.less_rebuild
    }
}

// spent runs f and adds the comparisons it made to `counter`.
fn spent<Q: Timers, T>(q: &mut Q, counter: &mut u64, f: impl FnOnce(&mut Q) -> T) -> T {
    let before = q.less_calls();
    let out = f(q);
    *counter += q.less_calls() - before;
    out
}

/// run replays the workload: each tick schedules an event due `horizon` ticks
/// later, delivers the cancels due now, then fires everything due (at <= now).
/// Cancels arrive at most 10 ticks after the schedule and every event is due
/// >= 1000 ticks out, so a cancel never meets an event that already fired.
fn run<Q: Timers>(q: &mut Q, mode: Mode, horizon: usize, w: &Workload) -> Res {
    let mut res = Res::default();
    let mut dead = vec![false; TICKS];
    let mut live = 0usize;
    for now in 0..TICKS {
        spent(q, &mut res.less_push, |q| q.push(now + horizon, now));
        live += 1;
        res.max_heap = res.max_heap.max(q.len());

        for &id in &w.cancel_at[now] {
            res.cancelled += 1;
            live -= 1;
            if mode == Mode::Fix {
                let removed = spent(q, &mut res.less_cancel, |q| q.remove(id));
                assert!(removed, "event {id} was not in the heap");
                continue;
            }
            dead[id] = true; // lazy: it stays in the heap
            if mode == Mode::LazyCompact && q.len() > 2 * live + 16 {
                res.compacted += spent(q, &mut res.less_rebuild, |q| q.compact(&dead));
                res.rebuilds += 1;
            }
        }

        while let Some(at) = q.peek_at()
            && at <= now
        {
            let id = spent(q, &mut res.less_pop, |q| q.pop()).expect("peek_at was Some");
            if dead[id] {
                res.stale_pops += 1;
                continue;
            }
            res.fired += 1;
            live -= 1;
            res.fired_ids.push(id);
        }
    }
    res.stale_left = q.dead_inside(&dead);
    res.valid = q.valid();
    res
}

const NAMES: [&str; 3] = ["fix (Remove)", "lazy", "lazy + compact"];
const MODES: [Mode; 3] = [Mode::Fix, Mode::Lazy, Mode::LazyCompact];

fn main() {
    println!("== (a) changing a priority while the item is in the heap ==");
    println!();
    part_a();

    println!();
    println!(
        "== (b) timers: 50,000 ticks, 9 in 10 events cancelled 1..10 ticks after they are scheduled =="
    );
    let mut r = Pcg::new(1, 2);
    let w = new_workload(&mut r);
    let delivered = w.cancelled.iter().filter(|&&c| c).count();
    println!("{TICKS} events scheduled, {delivered} cancels delivered before the run ends");

    for horizon in [1000, 100000] {
        println!();
        print!("every event due {horizon} ticks after it is scheduled");
        if horizon > TICKS {
            print!(" (longer than the run: nothing ever fires)");
        }
        println!();

        // The oracle: the events not cancelled whose due tick is inside the run,
        // in (at, id) order.
        let want: Vec<usize> = (0..TICKS)
            .filter(|&id| !w.cancelled[id] && id + horizon < TICKS)
            .collect();
        let res: Vec<Res> = MODES
            .iter()
            .map(|&m| run(&mut Heap::new(), m, horizon, &w))
            .collect();
        // std's BinaryHeap on the same workload: it has no Remove, so only the
        // lazy rows, and compaction is `retain`.
        let std_res: Vec<Res> = [Mode::Lazy, Mode::LazyCompact]
            .iter()
            .map(|&m| run(&mut StdTimers(BinaryHeap::new()), m, horizon, &w))
            .collect();
        println!(
            "  {:<15} {:>8} {:>7} {:>9} {:>10} {:>10} {:>11}",
            "strategy", "max heap", "fired", "cancelled", "stale pops", "compacted", "stale left"
        );
        for (name, x) in NAMES.iter().zip(&res) {
            println!(
                "  {:<15} {:>8} {:>7} {:>9} {:>10} {:>10} {:>11}",
                name, x.max_heap, x.fired, x.cancelled, x.stale_pops, x.compacted, x.stale_left
            );
        }
        println!(
            "  {:<15} {:>8} {:>10} {:>9} {:>9} {:>9} {:>11}",
            "strategy", "rebuilds", "Less total", "in Push", "in Pop", "in Remove", "in rebuilds"
        );
        let rows = NAMES.iter().chain(&["std lazy", "std lazy+retain"]);
        for (name, x) in rows.zip(res.iter().chain(&std_res)) {
            println!(
                "  {:<15} {:>8} {:>10} {:>9} {:>9} {:>9} {:>11}",
                name,
                x.rebuilds,
                x.less(),
                x.less_push,
                x.less_pop,
                x.less_cancel,
                x.less_rebuild
            );
        }
        let same = res.iter().all(|x| x.fired_ids == want);
        let valid = res.iter().all(|x| x.valid);
        // fix removes each cancelled event outright; every lazy cancel must be accounted for.
        let conserved = res[1..]
            .iter()
            .all(|x| x.cancelled == x.stale_pops + x.compacted + x.stale_left);
        println!(
            "  every strategy fired exactly the {} events the oracle expects, in order: {same}",
            want.len()
        );
        println!(
            "  lazy rows: cancelled == stale pops + compacted + stale left: {conserved}; heaps valid at the end: {valid}"
        );
        let (fix, lazy, compact) = (&res[0], &res[1], &res[2]);
        print!(
            "  lazy peaked at {:.1}x the heap that Remove needed ({} vs {} entries)",
            lazy.max_heap as f64 / fix.max_heap as f64,
            lazy.max_heap,
            fix.max_heap
        );
        println!(
            "; compaction at {:.1}x ({})",
            compact.max_heap as f64 / fix.max_heap as f64,
            compact.max_heap
        );
        println!(
            "  Less calls against fix: lazy {:.1}x, lazy + compact {:.1}x",
            lazy.less() as f64 / fix.less() as f64,
            compact.less() as f64 / fix.less() as f64
        );

        // Every count except Less must equal the own heap's: std changes how many
        // comparisons a pop or a rebuild makes, never which events fire or what is stored.
        let agree = std_res.iter().zip(&res[1..]).all(|(x, own)| {
            x.fired_ids == want
                && x.valid
                && (x.max_heap, x.fired, x.cancelled, x.stale_pops)
                    == (own.max_heap, own.fired, own.cancelled, own.stale_pops)
                && (x.compacted, x.stale_left, x.rebuilds)
                    == (own.compacted, own.stale_left, own.rebuilds)
        });
        println!(
            "  std rows: oracle's events fired, every count except Less equals the own heap's: {agree}"
        );
        println!(
            "  std Less against the own heap: lazy {:.2}x, lazy + retain {:.2}x",
            std_res[0].less() as f64 / lazy.less() as f64,
            std_res[1].less() as f64 / compact.less() as f64
        );
    }
}
```

**Rust output:**

```
== (a) changing a priority while the item is in the heap ==

  start: a1 ... f6       [a1 b2 c3 d4 e5 f6]  heap=true  pops: a(1) b(2) c(3) d(4) e(5) f(6)
  d.key = 0, no repair   [a1 b2 c3 d0 e5 f6]  heap=false pops: a(1) b(2) d(0) c(3) e(5) f(6)   <- not in priority order
  d.key = 0, rebuilt     [d0 a1 c3 b2 e5 f6]  heap=true  pops: d(0) a(1) b(2) c(3) e(5) f(6)
  retain(name != c)      [a1 b2 d4 e5 f6]  heap=true  pops: a(1) b(2) d(4) e(5) f(6)
  forgetting a written-through PeekMut: len 5 -> 1; pops: Some(0) None (the other four were leaked)

== (b) timers: 50,000 ticks, 9 in 10 events cancelled 1..10 ticks after they are scheduled ==
50000 events scheduled, 44964 cancels delivered before the run ends

every event due 1000 ticks after it is scheduled
  strategy        max heap   fired cancelled stale pops  compacted  stale left
  fix (Remove)         135    4916     44964          0          0           0
  lazy                1001    4916     44964      44084          0         880
  lazy + compact       276    4916     44964          0      44828         136
  strategy        rebuilds Less total   in Push    in Pop in Remove in rebuilds
  fix (Remove)           0     150089     49999     56745     43345           0
  lazy                   0     929776     49999    879777         0           0
  lazy + compact       377     150487     49999     62139         0       38349
  std lazy               0     538806     49999    488807         0           0
  std lazy+retain      377     123875     49999     35962         0       37914
  every strategy fired exactly the 4916 events the oracle expects, in order: true
  lazy rows: cancelled == stale pops + compacted + stale left: true; heaps valid at the end: true
  lazy peaked at 7.4x the heap that Remove needed (1001 vs 135 entries); compaction at 2.0x (276)
  Less calls against fix: lazy 6.2x, lazy + compact 1.0x
  std rows: oracle's events fired, every count except Less equals the own heap's: true
  std Less against the own heap: lazy 0.58x, lazy + retain 0.82x

every event due 100000 ticks after it is scheduled (longer than the run: nothing ever fires)
  strategy        max heap   fired cancelled stale pops  compacted  stale left
  fix (Remove)        5036       0     44964          0          0           0
  lazy               50000       0     44964          0          0       44964
  lazy + compact      9917       0     44964          0      44293         671
  strategy        rebuilds Less total   in Push    in Pop in Remove in rebuilds
  fix (Remove)           0      92542     49999         0     42543           0
  lazy                   0      49999     49999         0         0           0
  lazy + compact        46      93458     49999         0         0       43459
  std lazy               0      49999     49999         0         0           0
  std lazy+retain       46      55167     49999         0         0        5168
  every strategy fired exactly the 0 events the oracle expects, in order: true
  lazy rows: cancelled == stale pops + compacted + stale left: true; heaps valid at the end: true
  lazy peaked at 9.9x the heap that Remove needed (50000 vs 5036 entries); compaction at 2.0x (9917)
  Less calls against fix: lazy 0.5x, lazy + compact 1.0x
  std rows: oracle's events fired, every count except Less equals the own heap's: true
  std Less against the own heap: lazy 1.00x, lazy + retain 0.59x
```

> **Go vs Rust:** The own-heap tables are identical to Go's because `Heap` is `container/heap`'s `up`/`down`/`Remove`/`Init` ported line by line; it has to be hand-written, since `BinaryHeap` has no `Fix` or `Remove` (the repair in (a) is an O(n) `into_vec` + `from`, and a `Cell` is the only way safe Rust can change a key in place). The two `std` rows run the same workload through `BinaryHeap` with a counting `Ord` and differ in `Less` only: lazy 538,806 against 929,776, about 10 comparisons per pop against 18 over the 49,000 pops, which fits `pop` sinking a hole to the bottom with one comparison per level (`sift_down_to_bottom`) against Go's two per level (the mechanism read in the source, not run). At the 100,000-tick horizon `retain` compacted in 5,168 comparisons against 43,459 for filter + `Init`; `retain` re-sifts only from the first removed slot via `rebuild_tail` (source, plausible cause, not measured).

**Complexity:** `Fix`/`Remove` Θ(log n) with an index you maintain · lazy deletion: Θ(log n) push, Θ(1) cancel, one extra Θ(log n) pop per stale entry that surfaces · heap size up to live + stale · compaction Θ(n) each time, triggered at 2 × live + 16. Measured (Go, deterministic): with events due 1,000 ticks out, `Remove` peaks at 135 entries and 150,089 `Less` calls; lazy peaks at 1,001 entries and spends 929,776 `Less` calls (6.2x) on 44,084 stale pops, leaving 880 dead entries behind; lazy + compact peaks at 276 entries with 0 stale pops, 377 rebuilds and 150,487 `Less` calls. With events due 100,000 ticks out nothing ever surfaces: lazy makes 0 extra pops and the fewest `Less` calls of the three (49,999), but holds 50,000 entries, 44,964 of them dead; compaction caps it at 9,917 entries, about 2x the live count.

---

> Next tier: [🔴 hard](3-hard.md) — Huffman against the entropy bound, the Dijkstra frontier, what
> `container/heap` costs, a cheaper pop, and the capstone.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
