# Step 13 — Efficient Sorts: Divide & Conquer · 🔴 Hard

Examples **12–16**: McIlroy's adversary, introsort and the pdqsort tricks, an external sort, every
sort raced on comparisons and time, and the capstone.

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

> ← Back to [🟡 medium](2-medium.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. McIlroy's killer adversary

`🔴 hard` · *Quicksort / adversarial input*

Example 8 found a bad input for each fixed pivot rule by trying shapes. McIlroy's adversary builds one for any
quicksort without knowing its rule. It sorts the indices 0..n−1 with a comparator that holds the values back:
every element starts as "gas", undecided and above every decided value. When two gas elements meet, the
comparator must decide one of them. It freezes the one it suspects is the pivot (the gas element it saw
compared most recently) at the next smallest value. Every pivot then comes out near the bottom of its range,
and the split is about as lopsided as it can be. The answers never contradict each other, so the frozen values
are a real input: replay them with a plain comparator and the same deterministic sort makes the same
comparisons, Θ(n²) of them. A random-pivot quicksort is not hurt by that frozen input, because its pivot choices
do not depend on the input's order. Run live, though, the adversary beats the random pivot too, because it adapts
to whatever the sort compares. The program also attacks both standard-library sorts and prints the counts it gets. The attack is tuned for a
textbook quicksort, and against these sorts it backfires.

**Steps:**

1. Write the adversary as a comparator over indices. Write one Hoare quicksort that reaches the elements only
   through `cmp`, with median-of-3, ninther and random pivot rules.
2. Attack the median-of-3 sort, the ninther sort, `slices.SortFunc` and `slices.SortStableFunc` at n = 1,000 …
   16,000, and the random-pivot sort live. A quadratic sort grows 4× per doubling of n, an n log n sort a little
   over 2×.
3. Replay each frozen input on every sort, then time the median-of-3 killer at n = 50,000.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"time"
)

// ---------------------------------------------------------------------------
// McIlroy's adversary ("A Killer Adversary for Quicksort", 1999).
//
// The sort is handed the slice of INDICES 0..n-1 and a comparator. The
// comparator owns the values: val[i] is the value of element i. At the start
// every element is "gas" -- its value is undecided and larger than anything
// decided so far. Values are decided ("frozen", made "solid") lazily:
//
//   - when two gas elements meet, one of them must be frozen to answer. The
//     adversary freezes the one it suspects is the PIVOT (the "candidate"),
//     giving it the next-smallest solid value, so the pivot turns out tiny and
//     the partition is as lopsided as possible.
//   - the candidate is the gas element of the latest gas-vs-solid comparison.
//     A quicksort compares the pivot against everything, so the element that
//     keeps showing up in comparisons is the pivot.
//
// The answers form a consistent total order: a solid value never changes, and
// a newly frozen value is above every earlier solid and below every gas.
// So the final values are a real input, and replaying them with an ordinary
// comparator makes a deterministic sort repeat every one of its comparisons.
// ---------------------------------------------------------------------------

type adversary struct {
	val       []int
	gas       int // the value of every undecided element: n, above every solid
	nsolid    int // next solid value to hand out: 0, 1, 2, ...
	candidate int // the element we believe is the pivot
	ncmp      int
}

func newAdversary(n int) *adversary {
	a := &adversary{val: make([]int, n), gas: n}
	for i := range a.val {
		a.val[i] = a.gas
	}
	return a
}

func (a *adversary) cmp(x, y int) int {
	a.ncmp++
	if a.val[x] == a.gas && a.val[y] == a.gas {
		if x == a.candidate {
			a.val[x] = a.nsolid
		} else {
			a.val[y] = a.nsolid
		}
		a.nsolid++
	}
	if a.val[x] == a.gas {
		a.candidate = x
	} else if a.val[y] == a.gas {
		a.candidate = y
	}
	return cmp.Compare(a.val[x], a.val[y])
}

// attack runs sortFn on the identity permutation with the adversary as its
// comparator. It returns the comparison count and the frozen values: the
// concrete input that makes sortFn do exactly that work again.
func attack(n int, sortFn func([]int, func(a, b int) int)) (int, []int) {
	adv := newAdversary(n)
	idx := make([]int, n)
	for i := range idx {
		idx[i] = i
	}
	sortFn(idx, adv.cmp)
	// Check the claim above on every run: the sort's output is in order under
	// the FINAL values, and at most one element was never frozen (two gas
	// elements left adjacent would have had to be compared, freezing one).
	gas := 0
	for i, x := range idx {
		if adv.val[x] == adv.gas {
			gas++
		}
		if i > 0 && adv.val[idx[i-1]] > adv.val[x] {
			panic("adversary order was inconsistent")
		}
	}
	if gas > 1 {
		panic("more than one gas element left")
	}
	return adv.ncmp, adv.val
}

// shape describes an input: ascending runs, and inversions as a percentage of
// the n(n-1)/2 possible (0% = sorted, 50% = typical random, 100% = reversed).
func shape(xs []int) (runs int, invPct float64) {
	runs = 1
	inv := 0
	for i := range xs {
		if i > 0 && xs[i] < xs[i-1] {
			runs++
		}
		for _, y := range xs[i+1:] {
			if xs[i] > y {
				inv++
			}
		}
	}
	n := len(xs)
	return runs, 100 * float64(inv) / float64(n*(n-1)/2)
}

// ---------------------------------------------------------------------------
// The victims: one Hoare quicksort, three ways to pick its pivot.
// It touches elements ONLY through cmp, so it sorts indices for the adversary
// and plain ints for the replay with the same code.
// ---------------------------------------------------------------------------

type pivotRule func(xs []int, cmp func(a, b int) int) int

// median3 returns the index of the median of xs[a], xs[b], xs[c] with the
// three compare-exchanges Go's own source uses (order2 three times): always
// exactly 3 comparisons.
func median3(xs []int, a, b, c int, cmp func(a, b int) int) int {
	if cmp(xs[b], xs[a]) < 0 {
		a, b = b, a
	}
	if cmp(xs[c], xs[b]) < 0 {
		b, c = c, b
	}
	if cmp(xs[b], xs[a]) < 0 {
		a, b = b, a
	}
	return b
}

// pivotM3: median of first, middle and last.
func pivotM3(xs []int, cmp func(a, b int) int) int {
	n := len(xs)
	if n < 3 {
		return 0
	}
	return median3(xs, 0, n/2, n-1, cmp)
}

// pivotNinther: Tukey's ninther -- the median of three medians-of-3 taken
// from the start, the middle and the end -- for n >= 40; median-of-3 below.
func pivotNinther(xs []int, cmp func(a, b int) int) int {
	n := len(xs)
	if n < 40 {
		return pivotM3(xs, cmp)
	}
	s, m := n/8, n/2
	a := median3(xs, 0, s, 2*s, cmp)
	b := median3(xs, m-s, m, m+s, cmp)
	c := median3(xs, n-1-2*s, n-1-s, n-1, cmp)
	return median3(xs, a, b, c, cmp)
}

// pivotRandom: a uniformly random index (seeded PCG, so runs are repeatable).
func pivotRandom(r *rand.Rand) pivotRule {
	return func(xs []int, _ func(a, b int) int) int { return r.IntN(len(xs)) }
}

// quicksort: move the chosen pivot to the front, Hoare-partition around it
// (CLRS HOARE-PARTITION, written with i starting AT 0 so it ports to usize),
// recurse into the smaller side and loop on the larger -- the stack stays
// O(log n) even when the time is quadratic; only the time is under attack here.
func quicksort(xs []int, cmp func(a, b int) int, pick pivotRule) {
	for len(xs) > 1 {
		p := pick(xs, cmp)
		xs[0], xs[p] = xs[p], xs[0]
		pv := xs[0]
		i, j := 0, len(xs)
		for {
			for cmp(xs[i], pv) < 0 {
				i++
			}
			j--
			for cmp(xs[j], pv) > 0 {
				j--
			}
			if i >= j {
				break
			}
			xs[i], xs[j] = xs[j], xs[i]
			i++
		}
		// xs[:j+1] <= pv <= xs[j+1:], and both sides are non-empty
		left, right := xs[:j+1], xs[j+1:]
		if len(left) < len(right) {
			quicksort(left, cmp, pick)
			xs = right
		} else {
			quicksort(right, cmp, pick)
			xs = left
		}
	}
}

func victimM3(xs []int, c func(a, b int) int)      { quicksort(xs, c, pivotM3) }
func victimNinther(xs []int, c func(a, b int) int) { quicksort(xs, c, pivotNinther) }
func victimStd(xs []int, c func(a, b int) int)     { slices.SortFunc(xs, c) }
func victimStable(xs []int, c func(a, b int) int)  { slices.SortStableFunc(xs, c) }

// countOn sorts a copy of input with an ordinary counting comparator.
func countOn(input []int, sortFn func([]int, func(a, b int) int)) int {
	xs := slices.Clone(input)
	n := 0
	sortFn(xs, func(a, b int) int { n++; return cmp.Compare(a, b) })
	if !slices.IsSorted(xs) {
		panic("not sorted")
	}
	return n
}

func main() {
	type victim struct {
		name string
		fn   func([]int, func(a, b int) int)
	}
	victims := []victim{
		{"median-of-3", victimM3},
		{"ninther", victimNinther},
		{"SortFunc", victimStd},
		{"SortStableFunc", victimStable},
	}

	// 1) Attack each sort at doubling n. Quadratic work grows 4x per doubling,
	// n log n work a little over 2x.
	fmt.Println("1) the adversary against four sorts: comparisons (x = growth when n doubles)")
	fmt.Printf("%6s", "n")
	for _, v := range victims {
		fmt.Printf(" %14s %4s", v.name, "x")
	}
	fmt.Println()
	sizes := []int{1_000, 2_000, 4_000, 8_000, 16_000}
	last := make([]int, len(victims))
	for _, n := range sizes {
		fmt.Printf("%6d", n)
		for k, v := range victims {
			c, _ := attack(n, v.fn)
			if last[k] == 0 {
				fmt.Printf(" %14d %4s", c, "-")
			} else {
				fmt.Printf(" %14d %4.2f", c, float64(c)/float64(last[k]))
			}
			last[k] = c
		}
		fmt.Println()
	}
	nMax := sizes[len(sizes)-1]
	nf := float64(nMax)
	fmt.Printf("%-16s %8s %13s\n", fmt.Sprintf("at n = %d", nMax), "cmp/n^2", "cmp/(n lg n)")
	for k, v := range victims {
		c := float64(last[k])
		fmt.Printf("%-16s %8.3f %13.2f\n", v.name, c/(nf*nf), c/(nf*math.Log2(nf)))
	}
	fmt.Println("every run checked: output in order under the frozen values, at most 1 element left gas")

	// A random pivot defends against a FIXED input, not against a comparator
	// that adapts while the sort runs. Attacked live, it is quadratic too.
	// (Its own generator, so the replay below sees the same random stream.)
	live := rand.New(rand.NewPCG(3, 4))
	fmt.Print("random pivot, attacked live:")
	prevLive, lastLive := 0, 0
	for _, n := range sizes {
		c, _ := attack(n, func(xs []int, c func(a, b int) int) { quicksort(xs, c, pivotRandom(live)) })
		fmt.Printf(" %d", c)
		prevLive, lastLive = lastLive, c
	}
	nf2 := float64(nMax) * float64(nMax)
	fmt.Printf("\n  = %.3f n^2 at n = %d, x%.2f at the last doubling\n", float64(lastLive)/nf2, nMax, float64(lastLive)/float64(prevLive))

	// 2) Replay: the frozen values are an ordinary input.
	const n = 8_000
	fmt.Printf("\n2) replay at n = %d: the input frozen against each sort, sorted again with a plain\n", n)
	fmt.Println("   counting comparator by four sorts (and a random permutation for scale)")
	r := rand.New(rand.NewPCG(1, 2))
	cols := []victim{
		{"median-of-3", victimM3},
		{"ninther", victimNinther},
		{"random pivot", func(xs []int, c func(a, b int) int) { quicksort(xs, c, pivotRandom(r)) }},
		{"SortFunc", victimStd},
	}
	type row struct {
		victim string // the sort the input was frozen against ("" = random)
		input  []int
		live   int // comparisons during the attack, -1 for the random row
	}
	var rows []row
	for _, v := range victims {
		c, vals := attack(n, v.fn)
		rows = append(rows, row{v.name, vals, c})
	}
	perm := make([]int, n)
	for i := range perm {
		perm[i] = i
	}
	for i := n - 1; i > 0; i-- { // Fisher-Yates over IntN
		j := r.IntN(i + 1)
		perm[i], perm[j] = perm[j], perm[i]
	}
	rows = append(rows, row{"", perm, -1})

	fmt.Printf("%-18s %5s %6s %10s", "input frozen", "runs", "inv%", "live")
	for _, c := range cols {
		fmt.Printf(" %12s", c.name)
	}
	fmt.Println()
	replayMatches := 0
	for _, rw := range rows {
		runs, inv := shape(rw.input)
		name, live := "random permutation", "-"
		if rw.live >= 0 {
			name, live = "vs "+rw.victim, fmt.Sprint(rw.live)
		}
		fmt.Printf("%-18s %5d %6.2f %10s", name, runs, inv, live)
		for _, col := range cols {
			got := countOn(rw.input, col.fn)
			// the diagonal: the killer of a sort, replayed on that same sort
			if col.name == rw.victim && got == rw.live {
				replayMatches++
			}
			fmt.Printf(" %12d", got)
		}
		fmt.Println()
	}
	fmt.Printf("replay == live on the diagonal: %d of 3 (median-of-3, ninther, SortFunc)\n", replayMatches)

	// 3) The one timing table: the median-of-3 killer, replayed.
	const big = 50_000
	_, killer := attack(big, victimM3)
	fmt.Printf("\n3) time to sort the median-of-3 killer, n = %d (one run each)\n", big)
	timeIt := func(name string, f func([]int)) time.Duration {
		xs := slices.Clone(killer)
		t := time.Now()
		f(xs)
		d := time.Since(t)
		fmt.Printf("%-24s %10.2f ms\n", name, float64(d.Microseconds())/1000)
		return d
	}
	slow := timeIt("median-of-3 quicksort", func(xs []int) { quicksort(xs, cmp.Compare[int], pivotM3) })
	timeIt("random-pivot quicksort", func(xs []int) { quicksort(xs, cmp.Compare[int], pivotRandom(r)) })
	timeIt("slices.SortFunc", func(xs []int) { slices.SortFunc(xs, cmp.Compare[int]) })
	fast := timeIt("slices.Sort", func(xs []int) { slices.Sort(xs) })
	fmt.Printf("median-of-3 quicksort took %.0fx as long as slices.Sort on this input\n", float64(slow)/float64(fast))
}
```

**Sample output:**

```
1) the adversary against four sorts: comparisons (x = growth when n doubles)
     n    median-of-3    x        ninther    x       SortFunc    x SortStableFunc    x
  1000         255992    -         102608    -           7202    -           2129    -
  2000        1011992 3.95         392677 3.83          14215 1.97           4397 2.07
  4000        4023992 3.98        1535302 3.91          28246 1.99           8991 2.04
  8000       16047992 3.99        6070552 3.95          56259 1.99          18238 2.03
 16000       64095992 3.99       24141052 3.98         112290 2.00          36833 2.02
at n = 16000      cmp/n^2  cmp/(n lg n)
median-of-3         0.250        286.84
ninther             0.094        108.04
SortFunc            0.000          0.50
SortStableFunc      0.000          0.16
every run checked: output in order under the frozen values, at most 1 element left gas
random pivot, attacked live: 501498 2002998 8005998 32011998 128023998
  = 0.500 n^2 at n = 16000, x4.00 at the last doubling

2) replay at n = 8000: the input frozen against each sort, sorted again with a plain
   counting comparator by four sorts (and a random permutation for scale)
input frozen        runs   inv%       live  median-of-3      ninther random pivot     SortFunc
vs median-of-3      2000  33.31   16047992     16047992       175475       151185       101421
vs ninther          3538  47.33    6070552       200982      6070552       149003       121827
vs SortFunc            5   0.08      56259      2660419       149568       149726        56259
vs SortStableFunc    400   4.82      18238       177893       142752       158129       112292
random permutation  4034  50.17          -       158889       150340       142949       107392
replay == live on the diagonal: 3 of 3 (median-of-3, ninther, SortFunc)

3) time to sort the median-of-3 killer, n = 50000 (one run each)
median-of-3 quicksort        747.87 ms
random-pivot quicksort         2.47 ms
slices.SortFunc                1.46 ms
slices.Sort                    0.96 ms
median-of-3 quicksort took 782x as long as slices.Sort on this input
```

**Rust:**

```rust
use std::cmp::Ordering;
use std::time::{Duration, Instant};

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so the seeded
// random-pivot quicksort and the random permutation see Go's exact numbers.
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

// ---------------------------------------------------------------------------
// McIlroy's adversary ("A Killer Adversary for Quicksort", 1999).
//
// The sort is handed the INDICES 0..n-1 and a comparator. The comparator owns
// the values: val[i] is the value of element i. Every element starts as "gas":
// undecided, larger than anything decided so far. Values are frozen ("solid")
// lazily:
//   - when two gas elements meet, one must be frozen to answer. The adversary
//     freezes the one it suspects is the PIVOT (the "candidate") with the next
//     smallest solid value, so the pivot turns out tiny and the split lopsided.
//   - the candidate is the gas element of the latest gas-vs-solid comparison:
//     a quicksort compares its pivot against everything.
// A solid value never changes, and a new one is above every earlier solid and
// below every gas, so the answers form a consistent total order and the final
// values are a real input: replaying them makes a deterministic sort repeat
// every comparison.
// ---------------------------------------------------------------------------

struct Adversary {
    val: Vec<usize>,
    gas: usize,       // the value of every undecided element: n, above every solid
    nsolid: usize,    // next solid value to hand out: 0, 1, 2, ...
    candidate: usize, // the element we believe is the pivot
    ncmp: u64,
}

impl Adversary {
    fn new(n: usize) -> Self {
        Adversary {
            val: vec![n; n],
            gas: n,
            nsolid: 0,
            candidate: 0,
            ncmp: 0,
        }
    }

    // &mut self: the comparator MUTATES the adversary's state, so the closure
    // handed to a sort is an FnMut that borrows it mutably for the whole sort.
    fn cmp(&mut self, x: usize, y: usize) -> Ordering {
        self.ncmp += 1;
        if self.val[x] == self.gas && self.val[y] == self.gas {
            let frozen = if x == self.candidate { x } else { y };
            self.val[frozen] = self.nsolid;
            self.nsolid += 1;
        }
        if self.val[x] == self.gas {
            self.candidate = x;
        } else if self.val[y] == self.gas {
            self.candidate = y;
        }
        self.val[x].cmp(&self.val[y])
    }
}

// ---------------------------------------------------------------------------
// The victims: one Hoare quicksort, three ways to pick its pivot. It touches
// elements ONLY through cmp, so the same code sorts indices for the adversary
// and plain values for the replay.
// ---------------------------------------------------------------------------

enum Pivot<'a> {
    M3,
    Ninther,
    Random(&'a mut Pcg),
}

/// index of the median of xs[a], xs[b], xs[c] with the three
/// compare-exchanges Go's source uses (order2 three times): always 3 cmps.
fn median3<C: FnMut(usize, usize) -> Ordering>(
    xs: &[usize],
    (mut a, mut b, mut c): (usize, usize, usize),
    cmp: &mut C,
) -> usize {
    if cmp(xs[b], xs[a]).is_lt() {
        (a, b) = (b, a);
    }
    if cmp(xs[c], xs[b]).is_lt() {
        (b, c) = (c, b);
    }
    if cmp(xs[b], xs[a]).is_lt() {
        (a, b) = (b, a);
    }
    let _ = (a, c); // only the middle one is the answer
    b
}

fn pick<C: FnMut(usize, usize) -> Ordering>(xs: &[usize], cmp: &mut C, rule: &mut Pivot) -> usize {
    let n = xs.len();
    match rule {
        Pivot::Random(r) => r.int_n(n),
        _ if n < 3 => 0,
        // Tukey's ninther for n >= 40: the median of three medians-of-3 from
        // the start, the middle and the end
        Pivot::Ninther if n >= 40 => {
            let (s, m) = (n / 8, n / 2);
            let a = median3(xs, (0, s, 2 * s), cmp);
            let b = median3(xs, (m - s, m, m + s), cmp);
            let c = median3(xs, (n - 1 - 2 * s, n - 1 - s, n - 1), cmp);
            median3(xs, (a, b, c), cmp)
        }
        // median of first, middle, last (also the ninther's rule below 40)
        _ => median3(xs, (0, n / 2, n - 1), cmp),
    }
}

/// Move the pivot to the front, Hoare-partition around it (CLRS
/// HOARE-PARTITION with i starting AT 0, so no usize underflow), recurse into
/// the smaller side and loop on the larger: the stack stays O(log n) even when
/// the time is quadratic -- only the time is under attack here.
fn quicksort<C: FnMut(usize, usize) -> Ordering>(
    mut xs: &mut [usize],
    cmp: &mut C,
    rule: &mut Pivot,
) {
    while xs.len() > 1 {
        let p = pick(xs, cmp, rule);
        xs.swap(0, p);
        let pv = xs[0];
        let (mut i, mut j) = (0, xs.len());
        loop {
            while cmp(xs[i], pv).is_lt() {
                i += 1;
            }
            j -= 1;
            while cmp(xs[j], pv).is_gt() {
                j -= 1;
            }
            if i >= j {
                break;
            }
            xs.swap(i, j);
            i += 1;
        }
        // xs[..=j] <= pv <= xs[j+1..], both sides non-empty
        let (left, right) = std::mem::take(&mut xs).split_at_mut(j + 1);
        if left.len() < right.len() {
            quicksort(left, cmp, rule);
            xs = right;
        } else {
            quicksort(right, cmp, rule);
            xs = left;
        }
    }
}

#[derive(Clone, Copy, PartialEq)]
enum Victim {
    M3,
    Ninther,
    RandomPivot,
    Unstable, // slice::sort_unstable_by: ipnsort
    Stable,   // slice::sort_by: driftsort
}

impl Victim {
    fn name(self) -> &'static str {
        match self {
            Victim::M3 => "median-of-3",
            Victim::Ninther => "ninther",
            Victim::RandomPivot => "random pivot",
            Victim::Unstable => "sort_unstable_by",
            Victim::Stable => "sort_by",
        }
    }

    fn sort<C: FnMut(usize, usize) -> Ordering>(self, xs: &mut [usize], mut cmp: C, r: &mut Pcg) {
        match self {
            Victim::M3 => quicksort(xs, &mut cmp, &mut Pivot::M3),
            Victim::Ninther => quicksort(xs, &mut cmp, &mut Pivot::Ninther),
            Victim::RandomPivot => quicksort(xs, &mut cmp, &mut Pivot::Random(r)),
            Victim::Unstable => xs.sort_unstable_by(|a, b| cmp(*a, *b)),
            Victim::Stable => xs.sort_by(|a, b| cmp(*a, *b)),
        }
    }
}

/// Run the sort on the identity permutation with the adversary as comparator.
/// Returns the comparison count and the frozen values: the concrete input
/// that makes the sort do exactly that work again.
fn attack(n: usize, v: Victim, r: &mut Pcg) -> (u64, Vec<usize>) {
    let mut adv = Adversary::new(n);
    let mut idx: Vec<usize> = (0..n).collect();
    v.sort(&mut idx, |a, b| adv.cmp(a, b), r);
    // Check the claim above on every run: the output is in order under the
    // FINAL values, and at most one element was never frozen.
    let gas = idx.iter().filter(|&&x| adv.val[x] == adv.gas).count();
    assert!(
        idx.windows(2).all(|w| adv.val[w[0]] <= adv.val[w[1]]),
        "adversary order was inconsistent"
    );
    assert!(gas <= 1, "more than one gas element left");
    (adv.ncmp, adv.val)
}

/// ascending runs, and inversions as a percentage of the n(n-1)/2 possible
fn shape(xs: &[usize]) -> (usize, f64) {
    let runs = 1 + xs.windows(2).filter(|w| w[1] < w[0]).count();
    let inv: usize = (0..xs.len())
        .map(|i| xs[i + 1..].iter().filter(|&&y| xs[i] > y).count())
        .sum();
    let n = xs.len();
    (runs, 100.0 * inv as f64 / (n * (n - 1) / 2) as f64)
}

/// sort a copy of input with an ordinary counting comparator
fn count_on(input: &[usize], v: Victim, r: &mut Pcg) -> u64 {
    let mut xs = input.to_vec();
    let mut n = 0u64;
    v.sort(
        &mut xs,
        |a, b| {
            n += 1;
            a.cmp(&b)
        },
        r,
    );
    assert!(xs.is_sorted());
    n
}

fn main() {
    // the Go program's PCG only feeds the random pivot and the permutation;
    // the attack in section 1 never touches it, so one generator serves all.
    let mut r = Pcg::new(1, 2);
    let victims = [
        Victim::M3,
        Victim::Ninther,
        Victim::Unstable,
        Victim::Stable,
    ];

    // 1) Quadratic work grows 4x per doubling of n, n log n a little over 2x.
    println!("1) the adversary against four sorts: comparisons (x = growth when n doubles)");
    print!("{:>6}", "n");
    for v in victims {
        print!(" {:>16} {:>4}", v.name(), "x");
    }
    println!();
    let sizes = [1_000, 2_000, 4_000, 8_000, 16_000];
    let mut last = [0u64; 4];
    for n in sizes {
        print!("{n:>6}");
        for (k, v) in victims.into_iter().enumerate() {
            let (c, _) = attack(n, v, &mut r);
            if last[k] == 0 {
                print!(" {c:>16} {:>4}", "-");
            } else {
                print!(" {c:>16} {:>4.2}", c as f64 / last[k] as f64);
            }
            last[k] = c;
        }
        println!();
    }
    let n_max = sizes[sizes.len() - 1];
    let nf = n_max as f64;
    println!(
        "{:<16} {:>8} {:>13}",
        format!("at n = {n_max}"),
        "cmp/n^2",
        "cmp/(n lg n)"
    );
    for (k, v) in victims.into_iter().enumerate() {
        let c = last[k] as f64;
        println!(
            "{:<16} {:>8.3} {:>13.2}",
            v.name(),
            c / (nf * nf),
            c / (nf * nf.log2())
        );
    }
    println!(
        "every run checked: output in order under the frozen values, at most 1 element left gas"
    );
    println!(
        "(and neither std sort panicked: the adversary's answers are a consistent total order)"
    );

    // A random pivot defends against a FIXED input, not against a comparator
    // that adapts while the sort runs. Attacked live, it is quadratic too.
    // (Its own generator, so the replay below sees the same random stream.)
    let mut live = Pcg::new(3, 4);
    print!("random pivot, attacked live:");
    let (mut prev_live, mut last_live) = (0, 0);
    for n in sizes {
        let (c, _) = attack(n, Victim::RandomPivot, &mut live);
        print!(" {c}");
        (prev_live, last_live) = (last_live, c);
    }
    println!(
        "\n  = {:.3} n^2 at n = {n_max}, x{:.2} at the last doubling",
        last_live as f64 / (nf * nf),
        last_live as f64 / prev_live as f64
    );

    // 2) Replay: the frozen values are an ordinary input.
    const N: usize = 8_000;
    println!(
        "\n2) replay at n = {N}: the input frozen against each sort, sorted again with a plain"
    );
    println!("   counting comparator by four sorts (and a random permutation for scale)");
    let cols = [
        Victim::M3,
        Victim::Ninther,
        Victim::RandomPivot,
        Victim::Unstable,
    ];
    // (the sort it was frozen against, the input, comparisons during the attack)
    let mut rows: Vec<(Option<Victim>, Vec<usize>, u64)> = victims
        .into_iter()
        .map(|v| {
            let (c, vals) = attack(N, v, &mut r);
            (Some(v), vals, c)
        })
        .collect();
    let mut perm: Vec<usize> = (0..N).collect();
    for i in (1..N).rev() {
        // Fisher-Yates over IntN, as in Go
        let j = r.int_n(i + 1);
        perm.swap(i, j);
    }
    rows.push((None, perm, 0));

    print!(
        "{:<20} {:>5} {:>6} {:>10}",
        "input frozen", "runs", "inv%", "live"
    );
    for c in cols {
        print!(" {:>16}", c.name());
    }
    println!();
    let mut replay_matches = 0;
    for (victim, input, live) in &rows {
        let (runs, inv) = shape(input);
        let (name, live_s) = match victim {
            Some(v) => (format!("vs {}", v.name()), live.to_string()),
            None => ("random permutation".to_string(), "-".to_string()),
        };
        print!("{name:<20} {runs:>5} {inv:>6.2} {live_s:>10}");
        for col in cols {
            let got = count_on(input, col, &mut r);
            // the diagonal: the killer of a sort, replayed on that same sort
            if *victim == Some(col) && got == *live {
                replay_matches += 1;
            }
            print!(" {got:>16}");
        }
        println!();
    }
    println!(
        "replay == live on the diagonal: {replay_matches} of 3 (median-of-3, ninther, sort_unstable_by)"
    );

    // 3) The one timing table: the median-of-3 killer, replayed.
    const BIG: usize = 50_000;
    let (_, killer) = attack(BIG, Victim::M3, &mut r);
    println!("\n3) time to sort the median-of-3 killer, n = {BIG} (one run each)");
    let time_it = |name: &str, f: &mut dyn FnMut(&mut [usize])| -> Duration {
        let mut xs = killer.clone();
        let t = Instant::now();
        f(&mut xs);
        let d = t.elapsed();
        assert!(xs.is_sorted());
        println!("{name:<24} {:>10.2} ms", d.as_micros() as f64 / 1000.0);
        d
    };
    let slow = time_it("median-of-3 quicksort", &mut |xs| {
        quicksort(xs, &mut |a: usize, b: usize| a.cmp(&b), &mut Pivot::M3)
    });
    time_it("random-pivot quicksort", &mut |xs| {
        quicksort(
            xs,
            &mut |a: usize, b: usize| a.cmp(&b),
            &mut Pivot::Random(&mut r),
        )
    });
    time_it("sort_unstable_by", &mut |xs| {
        xs.sort_unstable_by(usize::cmp)
    });
    let fast = time_it("sort_unstable", &mut |xs| xs.sort_unstable());
    println!(
        "median-of-3 quicksort took {:.0}x as long as sort_unstable on this input",
        slow.as_secs_f64() / fast.as_secs_f64()
    );
}
```

**Rust sample output:**

```
1) the adversary against four sorts: comparisons (x = growth when n doubles)
     n      median-of-3    x          ninther    x sort_unstable_by    x          sort_by    x
  1000           255992    -           102608    -              999    -              999    -
  2000          1011992 3.95           392677 3.83             1999 2.00             1999 2.00
  4000          4023992 3.98          1535302 3.91             3999 2.00             3999 2.00
  8000         16047992 3.99          6070552 3.95             7999 2.00             7999 2.00
 16000         64095992 3.99         24141052 3.98            15999 2.00            15999 2.00
at n = 16000      cmp/n^2  cmp/(n lg n)
median-of-3         0.250        286.84
ninther             0.094        108.04
sort_unstable_by    0.000          0.07
sort_by             0.000          0.07
every run checked: output in order under the frozen values, at most 1 element left gas
(and neither std sort panicked: the adversary's answers are a consistent total order)
random pivot, attacked live: 501498 2002998 8005998 32011998 128023998
  = 0.500 n^2 at n = 16000, x4.00 at the last doubling

2) replay at n = 8000: the input frozen against each sort, sorted again with a plain
   counting comparator by four sorts (and a random permutation for scale)
input frozen          runs   inv%       live      median-of-3          ninther     random pivot sort_unstable_by
vs median-of-3        2000  33.31   16047992         16047992           175475           151185           117144
vs ninther            3538  47.33    6070552           200982          6070552           149003           107778
vs sort_unstable_by      1   0.00       7999           128187           130482           145435             7999
vs sort_by               1   0.00       7999           128187           130482           153472             7999
random permutation    4034  50.17          -           158889           150340           142949           109230
replay == live on the diagonal: 3 of 3 (median-of-3, ninther, sort_unstable_by)

3) time to sort the median-of-3 killer, n = 50000 (one run each)
median-of-3 quicksort        197.91 ms
random-pivot quicksort         1.50 ms
sort_unstable_by               0.43 ms
sort_unstable                  0.43 ms
median-of-3 quicksort took 463x as long as sort_unstable on this input
```

> **Go vs Rust:** The hand-written rows (median-of-3, ninther, the live random-pivot line, and the replay's random pivot and random permutation) are identical to Go's. The standard-library rows differ, and the reason can be read in the source (not instrumented here). Both Rust sorts begin with `find_existing_run`, which compares v[1] with v[0], then v[2] with v[1], and so on. At each step the adversary freezes the previous element at the next value, so it builds a sorted input by itself, and `sort_unstable_by` and `sort_by` finish in exactly n − 1 comparisons without partitioning anything. Go's pdqsort samples pivots first and made 0.50 n lg n. Neither Rust sort panicked; a panic is allowed only for an inconsistent order, and these answers are consistent. On the timing line the hand-written quadratic sort was faster in Rust than in Go in every run, by roughly 2–4× (150–350 ms against 600–850 ms across runs), and it was still a few hundred times slower than `sort_unstable`.

**Complexity:** a fixed-rule quicksort is Θ(n²) on the input frozen against it: 0.250 n² comparisons for median-of-3 and 0.094 n² for the ninther at n = 16,000, growing ×3.99 and ×3.98 per doubling, and a replay repeats the count exactly. A random pivot sorts the same input in about 150,000 comparisons at n = 8,000, the cost of a random permutation. Attacked live, the same random-pivot sort makes 0.500 n² comparisons, ×4.00 per doubling: a random pivot protects against a fixed input, not against a comparator that adapts. The attack backfires on the standard library. `slices.SortFunc` grows ×2.00 per doubling (0.50 n lg n at n = 16,000), fewer comparisons than it spends on a random permutation, and the input frozen against it is nearly sorted (5 ascending runs). Its heapsort fallback never ran: an instrumented copy of Go's source, not shown, gives the same counts and zero heapsort calls. On the replayed killer at n = 50,000 the median-of-3 quicksort took hundreds of times as long as `slices.Sort` in every run. The exact ratio swings with the load on this shared machine (from under 500× to over 1,000×), and the printed one is a single run.

---

## 13. Introsort, then the pdqsort tricks

`🔴 hard` · *Hybrid sort / Θ(n log n) worst case*

A standard-library sort is easier to read when you build it yourself, one layer at a time, and count what each
layer buys. v0 is example 12's median-of-3 Hoare quicksort. v1 gives it a depth budget of 2⌊lg n⌋ partition
levels and heapsorts any range that runs out. That is Musser's introsort, and the budget alone turns Θ(n²) into
Θ(n log n) worst case. v2 hands ranges of 12 or fewer to insertion sort, which saves comparisons on every input.
v3 adds three tricks from Go's pdqsort source. If every compare-exchange of the pivot sample swapped, it reverses
the range. If the sample was in order and the last partition moved nothing, it runs a partial insertion sort,
limited to 5 fixes, and stops if that sorts the range. If the element before the range is not less than the
pivot, it splits off the block equal to the pivot and never touches it again. Each layer runs on eight inputs,
including McIlroy killers generated against v0, v1 and v3. A second table counts how often the heapsort
fallback fired and how many elements it got.

**Steps:**

1. Write one `sort` method that switches on its layer (0–3), with counters for comparisons, heapsort fallbacks
   and the elements those fallbacks sorted.
2. Generate the inputs (random, sorted, reversed, all-equal, organ-pipe and three adversary inputs) at n = 50,000
   and count every layer plus `slices.SortFunc` on each.
3. Read the tables column by column. Each layer fixes some row, and the fallback table shows where heapsort did
   the work.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
)

// One quicksort, built up one layer at a time:
//
//	v0  median-of-3 Hoare quicksort (example 12's victim)
//	v1  + a depth budget of 2*floor(lg n) levels, then heapsort (Musser's introsort)
//	v2  + insertion sort for ranges of length <= 12
//	v3  + three tricks from Go's pdqsort (src/slices/zsortordered.go):
//	      reverse a range whose pivot sample was strictly descending,
//	      a bounded partial insertion sort when the range looks sorted,
//	      an equal-key partition when the pivot equals the element before the range.
//
// Every version sorts through cmp only, so McIlroy's adversary can attack it.

type sorter struct {
	cmp       func(a, b int) int
	layer     int // 0..3
	fallbacks int // ranges handed to heapsort
	heapElems int // total length of those ranges
}

// median3 returns the index of the median of xs[a], xs[b], xs[c] with three
// compare-exchanges (Go's order2 x3) and reports how many of them swapped:
// 0 = the sample was increasing, 3 = strictly decreasing.
func (s *sorter) median3(xs []int, a, b, c int) (int, int) {
	swaps := 0
	if s.cmp(xs[b], xs[a]) < 0 {
		a, b = b, a
		swaps++
	}
	if s.cmp(xs[c], xs[b]) < 0 {
		b, c = c, b
		swaps++
	}
	if s.cmp(xs[b], xs[a]) < 0 {
		a, b = b, a
		swaps++
	}
	return b, swaps
}

func (s *sorter) insertionSort(xs []int) {
	for i := 1; i < len(xs); i++ {
		for j := i; j > 0 && s.cmp(xs[j], xs[j-1]) < 0; j-- {
			xs[j], xs[j-1] = xs[j-1], xs[j]
		}
	}
}

// heapSort: Θ(n log n) worst case, in place -- the guarantee introsort buys.
func (s *sorter) heapSort(xs []int) {
	siftDown := func(root, n int) {
		for {
			child := 2*root + 1
			if child >= n {
				return
			}
			if child+1 < n && s.cmp(xs[child], xs[child+1]) < 0 {
				child++
			}
			if s.cmp(xs[root], xs[child]) >= 0 {
				return
			}
			xs[root], xs[child] = xs[child], xs[root]
			root = child
		}
	}
	for i := (len(xs) - 1) / 2; i >= 0; i-- {
		siftDown(i, len(xs))
	}
	for end := len(xs) - 1; end > 0; end-- {
		xs[0], xs[end] = xs[end], xs[0]
		siftDown(0, end)
	}
}

// partialInsertionSort (Go's, same constants): fix at most 5 adjacent
// out-of-order pairs, only on ranges of 50 or more; true if that sorted it.
func (s *sorter) partialInsertionSort(xs []int) bool {
	const maxSteps, shortestShifting = 5, 50
	i := 1
	for step := 0; step < maxSteps; step++ {
		for i < len(xs) && s.cmp(xs[i], xs[i-1]) >= 0 {
			i++
		}
		if i == len(xs) {
			return true
		}
		if len(xs) < shortestShifting {
			return false
		}
		xs[i], xs[i-1] = xs[i-1], xs[i]
		for j := i - 1; j > 0 && s.cmp(xs[j], xs[j-1]) < 0; j-- { // smaller one left
			xs[j], xs[j-1] = xs[j-1], xs[j]
		}
		for j := i + 1; j < len(xs) && s.cmp(xs[j], xs[j-1]) < 0; j++ { // larger one right
			xs[j], xs[j-1] = xs[j-1], xs[j]
		}
	}
	return false
}

// partitionEqual (Go's): the pivot (moved to xs[0]) is the minimum of the
// range, so split it into "== pivot" and "> pivot" and return where the
// second block starts. The first block is finished.
func (s *sorter) partitionEqual(xs []int) int {
	i, j := 1, len(xs)-1
	for {
		for i <= j && s.cmp(xs[0], xs[i]) >= 0 {
			i++
		}
		for i <= j && s.cmp(xs[0], xs[j]) < 0 {
			j--
		}
		if i > j {
			return i
		}
		xs[i], xs[j] = xs[j], xs[i]
		i++
		j--
	}
}

// sort is the whole hybrid. limit is the remaining depth budget (v1+); pred
// is the element just before this range, if any (every element of the range
// is >= it). wasBalanced/wasPartitioned describe the previous partition, as
// in Go's pdqsort.
func (s *sorter) sort(xs []int, limit int, pred *int) {
	wasBalanced, wasPartitioned := true, true
	for len(xs) > 1 {
		if s.layer >= 2 && len(xs) <= 12 {
			s.insertionSort(xs)
			return
		}
		if s.layer >= 1 {
			if limit == 0 {
				s.fallbacks++
				s.heapElems += len(xs)
				s.heapSort(xs)
				return
			}
			limit--
		}

		n := len(xs)
		p, swaps := 0, 0
		if n >= 3 {
			p, swaps = s.median3(xs, 0, n/2, n-1)
		}
		sortedHint := swaps == 0
		if s.layer >= 3 && n >= 3 && swaps == 3 {
			// Every compare-exchange swapped: the sample was strictly
			// descending. Guess the whole range is, reverse it, and treat it
			// as ascending (the pivot index moves with it).
			slices.Reverse(xs)
			p = n - 1 - p
			sortedHint = true
		}
		if s.layer >= 3 && wasBalanced && wasPartitioned && sortedHint {
			if s.partialInsertionSort(xs) {
				return
			}
		}

		xs[0], xs[p] = xs[p], xs[0]
		pv := xs[0]
		if s.layer >= 3 && pred != nil && s.cmp(*pred, pv) >= 0 {
			// pred <= everything here, and pred >= pivot: the pivot is the
			// range minimum. Park the equal block; never touch it again.
			xs = xs[s.partitionEqual(xs):]
			continue
		}

		// Hoare partition around xs[0] (CLRS, with i starting at 0).
		i, j := 0, n
		swapped := false
		for {
			for s.cmp(xs[i], pv) < 0 {
				i++
			}
			j--
			for s.cmp(xs[j], pv) > 0 {
				j--
			}
			if i >= j {
				break
			}
			xs[i], xs[j] = xs[j], xs[i]
			swapped = swapped || i > 0 // the pivot's own move to the left side is not evidence
			i++
		}
		left, right := xs[:j+1], xs[j+1:]
		wasBalanced = min(len(left), len(right)) >= n/8
		wasPartitioned = !swapped

		// Recurse into the smaller side, loop on the larger: stack <= lg n.
		// Everything on the right is >= pv, so pv is the right side's pred.
		if len(left) < len(right) {
			s.sort(left, limit, pred)
			xs, pred = right, &pv
		} else {
			s.sort(right, limit, &pv)
			xs = left
		}
	}
}

func run(input []int, layer int, c func(a, b int) int) {
	s := &sorter{cmp: c, layer: layer}
	s.sort(slices.Clone(input), 2*(bits.Len(uint(len(input)))-1), nil)
}

// fallback is how often the depth budget ran out, and on how many elements.
type fallback struct{ ranges, elems int }

// count sorts a copy with a plain counting comparator; checks the result.
func count(input []int, layer int) (cmps int, fb fallback) {
	xs := slices.Clone(input)
	s := &sorter{cmp: func(a, b int) int { cmps++; return cmp.Compare(a, b) }, layer: layer}
	s.sort(xs, 2*(bits.Len(uint(len(xs)))-1), nil)
	if !slices.IsSorted(xs) {
		panic(fmt.Sprintf("v%d did not sort", layer))
	}
	return cmps, fallback{s.fallbacks, s.heapElems}
}

// killer runs McIlroy's adversary (example 12) against one layer and returns
// the frozen values: an input that makes that layer repeat its work.
func killer(n, layer int) []int {
	val := make([]int, n)
	gas, nsolid, candidate := n, 0, 0
	for i := range val {
		val[i] = gas
	}
	idx := make([]int, n)
	for i := range idx {
		idx[i] = i
	}
	run(idx, layer, func(x, y int) int {
		if val[x] == gas && val[y] == gas {
			if x == candidate {
				val[x] = nsolid
			} else {
				val[y] = nsolid
			}
			nsolid++
		}
		if val[x] == gas {
			candidate = x
		} else if val[y] == gas {
			candidate = y
		}
		return cmp.Compare(val[x], val[y])
	})
	return val
}

func main() {
	// n = 50,000: v0 is quadratic on the killer inputs (several hundred
	// million comparisons each), and every killer is generated by one more
	// such run; at 100,000 that work would be four times larger.
	const n = 50_000
	r := rand.New(rand.NewPCG(1, 2))
	random := make([]int, n)
	sorted := make([]int, n)
	reversed := make([]int, n)
	equal := make([]int, n)
	organ := make([]int, n)
	for i := range n {
		random[i], sorted[i], reversed[i], equal[i] = i, i, n-1-i, 7
		organ[i] = min(i, n-1-i) // 0,1,...,n/2-1, n/2-1,...,0
	}
	for i := n - 1; i > 0; i-- { // Fisher-Yates over IntN
		j := r.IntN(i + 1)
		random[i], random[j] = random[j], random[i]
	}
	inputs := []struct {
		name string
		xs   []int
	}{
		{"random", random}, {"sorted", sorted}, {"reversed", reversed},
		{"all-equal", equal}, {"organ-pipe", organ},
		{"killer of v0", killer(n, 0)}, {"killer of v1", killer(n, 1)},
		{"killer of v3", killer(n, 3)},
	}

	// Count everything once: 4 layers + the standard library per input.
	cmps := make([][5]int, len(inputs))
	falls := make([][4]fallback, len(inputs))
	for k, in := range inputs {
		for layer := range 4 {
			cmps[k][layer], falls[k][layer] = count(in.xs, layer)
		}
		slices.SortFunc(slices.Clone(in.xs), func(a, b int) int { cmps[k][4]++; return cmp.Compare(a, b) })
	}

	lg := math.Log2(n)
	budget := 2 * (bits.Len(uint(n)) - 1)
	fmt.Printf("n = %d, n lg n = %.0f, depth budget 2*floor(lg n) = %d\n\n", n, n*lg, budget)
	header := func(title string) {
		fmt.Printf("%-14s %10s %10s %10s %10s %10s", title, "v0 quick", "v1 +heap", "v2 +ins", "v3 +pdq", "SortFunc")
	}
	header("comparisons")
	fmt.Println()
	for k, in := range inputs {
		fmt.Printf("%-14s", in.name)
		for _, c := range cmps[k] {
			fmt.Printf(" %10d", c)
		}
		fmt.Println()
	}

	fmt.Printf("\n%-14s %21s %21s %21s\n", "heapsort", "v1 +heap", "v2 +ins", "v3 +pdq")
	fmt.Printf("%-14s %21s %21s %21s\n", "fallbacks", "ranges / elements", "ranges / elements", "ranges / elements")
	for k, in := range inputs {
		fmt.Printf("%-14s", in.name)
		for _, f := range falls[k][1:] {
			fmt.Printf(" %10d / %8d", f.ranges, f.elems)
		}
		fmt.Println()
	}

	fmt.Println()
	header("/ (n lg n)")
	fmt.Println()
	for k, in := range inputs {
		fmt.Printf("%-14s", in.name)
		for _, c := range cmps[k] {
			fmt.Printf(" %10.2f", float64(c)/(n*lg))
		}
		fmt.Println()
	}
}
```

**Output:**

```
n = 50000, n lg n = 780482, depth budget 2*floor(lg n) = 30

comparisons      v0 quick   v1 +heap    v2 +ins    v3 +pdq   SortFunc
random            1172008    1170011     981276     960775     820057
sorted             965531     965531     680795      50002      50011
reversed           990521     990521     705781      50002      50011
all-equal          963323     963323     677739      50002      50011
organ-pipe        3002686    2278754    2268842    2464221     756354
killer of v0    625299992    2891928    2891847    2124361     752853
killer of v1    417584421    2863573    2863492    2848506    1536441
killer of v3    313220244    1959373    1816993      75003     125032

heapsort                    v1 +heap               v2 +ins               v3 +pdq
fallbacks          ranges / elements     ranges / elements     ranges / elements
random                192 /      747          5 /       86          0 /        0
sorted                  0 /        0          0 /        0          0 /        0
reversed                0 /        0          0 /        0          0 /        0
all-equal               0 /        0          0 /        0          0 /        0
organ-pipe            496 /    47714        149 /    46382        196 /    44203
killer of v0            2 /    49944          1 /    49942         46 /    32506
killer of v1            2 /    49944          1 /    49942          2 /    49787
killer of v3            1 /    24972          1 /    24972          0 /        0

/ (n lg n)       v0 quick   v1 +heap    v2 +ins    v3 +pdq   SortFunc
random               1.50       1.50       1.26       1.23       1.05
sorted               1.24       1.24       0.87       0.06       0.06
reversed             1.27       1.27       0.90       0.06       0.06
all-equal            1.23       1.23       0.87       0.06       0.06
organ-pipe           3.85       2.92       2.91       3.16       0.97
killer of v0       801.17       3.71       3.71       2.72       0.96
killer of v1       535.03       3.67       3.67       3.65       1.97
killer of v3       401.32       2.51       2.33       0.10       0.16
```

**Rust:**

```rust
use std::cmp::Ordering;

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so the random
// permutation is the one the Go program sorts.
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

// One quicksort, built up one layer at a time:
//   v0  median-of-3 Hoare quicksort (example 12's victim)
//   v1  + a depth budget of 2*floor(lg n) levels, then heapsort (Musser's introsort)
//   v2  + insertion sort for ranges of length <= 12
//   v3  + three tricks from Go's pdqsort (src/slices/zsortordered.go):
//         reverse a range whose pivot sample was strictly descending,
//         a bounded partial insertion sort when the range looks sorted,
//         an equal-key partition when the pivot equals the element before the range.
// Every version sorts through cmp only, so McIlroy's adversary can attack it.
// The comparator is a generic FnMut: monomorphised, and free to mutate state.

struct Sorter<C: FnMut(usize, usize) -> Ordering> {
    cmp: C,
    layer: u8,         // 0..=3
    fallbacks: usize,  // ranges handed to heapsort
    heap_elems: usize, // total length of those ranges
}

impl<C: FnMut(usize, usize) -> Ordering> Sorter<C> {
    fn less(&mut self, a: usize, b: usize) -> bool {
        (self.cmp)(a, b).is_lt()
    }

    /// index of the median of xs[a], xs[b], xs[c] with three compare-exchanges
    /// (Go's order2 x3), and how many swapped: 0 = increasing sample,
    /// 3 = strictly decreasing.
    fn median3(&mut self, xs: &[usize], mut a: usize, mut b: usize, mut c: usize) -> (usize, u8) {
        let mut swaps = 0;
        if self.less(xs[b], xs[a]) {
            (a, b) = (b, a);
            swaps += 1;
        }
        if self.less(xs[c], xs[b]) {
            (b, c) = (c, b);
            swaps += 1;
        }
        if self.less(xs[b], xs[a]) {
            (a, b) = (b, a);
            swaps += 1;
        }
        let _ = (a, c); // only the middle one is the answer
        (b, swaps)
    }

    fn insertion_sort(&mut self, xs: &mut [usize]) {
        for i in 1..xs.len() {
            let mut j = i;
            while j > 0 && self.less(xs[j], xs[j - 1]) {
                xs.swap(j, j - 1);
                j -= 1;
            }
        }
    }

    /// Θ(n log n) worst case, in place: the guarantee introsort buys.
    fn heap_sort(&mut self, xs: &mut [usize]) {
        fn sift_down<C: FnMut(usize, usize) -> Ordering>(
            s: &mut Sorter<C>,
            xs: &mut [usize],
            mut root: usize,
            n: usize,
        ) {
            loop {
                let mut child = 2 * root + 1;
                if child >= n {
                    return;
                }
                if child + 1 < n && s.less(xs[child], xs[child + 1]) {
                    child += 1;
                }
                if !s.less(xs[root], xs[child]) {
                    return;
                }
                xs.swap(root, child);
                root = child;
            }
        }
        let n = xs.len();
        for i in (0..n / 2).rev() {
            // Go starts at (n-1)/2; for n >= 2 that node is a leaf or the
            // same node, and a leaf's sift-down makes no comparison
            sift_down(self, xs, i, n);
        }
        for end in (1..n).rev() {
            xs.swap(0, end);
            sift_down(self, xs, 0, end);
        }
    }

    /// Go's partialInsertionSort, same constants: fix at most 5 adjacent
    /// out-of-order pairs, only on ranges of 50 or more; true if that sorted it.
    fn partial_insertion_sort(&mut self, xs: &mut [usize]) -> bool {
        const MAX_STEPS: usize = 5;
        const SHORTEST_SHIFTING: usize = 50;
        let n = xs.len();
        let mut i = 1;
        for _ in 0..MAX_STEPS {
            while i < n && !self.less(xs[i], xs[i - 1]) {
                i += 1;
            }
            if i == n {
                return true;
            }
            if n < SHORTEST_SHIFTING {
                return false;
            }
            xs.swap(i, i - 1);
            let mut j = i - 1; // smaller one left
            while j > 0 && self.less(xs[j], xs[j - 1]) {
                xs.swap(j, j - 1);
                j -= 1;
            }
            let mut j = i + 1; // larger one right
            while j < n && self.less(xs[j], xs[j - 1]) {
                xs.swap(j, j - 1);
                j += 1;
            }
        }
        false
    }

    /// Go's partitionEqual: the pivot (at xs[0]) is the range minimum, so split
    /// into "== pivot" and "> pivot"; returns where the second block starts.
    fn partition_equal(&mut self, xs: &mut [usize]) -> usize {
        let (mut i, mut j) = (1, xs.len() - 1);
        loop {
            while i <= j && !self.less(xs[0], xs[i]) {
                i += 1;
            }
            while i <= j && self.less(xs[0], xs[j]) {
                j -= 1; // i <= j and i >= 1, so j >= 1 here: no underflow
            }
            if i > j {
                return i;
            }
            xs.swap(i, j);
            i += 1;
            j -= 1;
        }
    }

    /// The whole hybrid. limit: remaining depth budget (v1+). pred: the element
    /// just before this range, if any (every element here is >= it).
    fn sort(&mut self, mut xs: &mut [usize], mut limit: u32, mut pred: Option<usize>) {
        let (mut was_balanced, mut was_partitioned) = (true, true);
        while xs.len() > 1 {
            if self.layer >= 2 && xs.len() <= 12 {
                self.insertion_sort(xs);
                return;
            }
            if self.layer >= 1 {
                if limit == 0 {
                    self.fallbacks += 1;
                    self.heap_elems += xs.len();
                    self.heap_sort(xs);
                    return;
                }
                limit -= 1;
            }

            let n = xs.len();
            let (mut p, swaps) = if n >= 3 {
                self.median3(xs, 0, n / 2, n - 1)
            } else {
                (0, 0)
            };
            let mut sorted_hint = swaps == 0;
            if self.layer >= 3 && swaps == 3 {
                // Every compare-exchange swapped: the sample was strictly
                // descending. Reverse the range, treat it as ascending.
                xs.reverse();
                p = n - 1 - p;
                sorted_hint = true;
            }
            if self.layer >= 3
                && was_balanced
                && was_partitioned
                && sorted_hint
                && self.partial_insertion_sort(xs)
            {
                return;
            }

            xs.swap(0, p);
            let pv = xs[0];
            if self.layer >= 3
                && let Some(pr) = pred
                && !self.less(pr, pv)
            {
                // pred <= everything here and pred >= pivot: the pivot is the
                // range minimum. Park the equal block; never touch it again.
                let mid = self.partition_equal(xs);
                xs = &mut xs[mid..];
                continue;
            }

            // Hoare partition around xs[0] (CLRS, with i starting at 0).
            let (mut i, mut j) = (0, n);
            let mut swapped = false;
            loop {
                while self.less(xs[i], pv) {
                    i += 1;
                }
                j -= 1;
                // (self.cmp)(xs[j], pv), not less(pv, xs[j]): same count, but
                // the adversary's answer depends on the argument order
                while (self.cmp)(xs[j], pv).is_gt() {
                    j -= 1;
                }
                if i >= j {
                    break;
                }
                xs.swap(i, j);
                swapped = swapped || i > 0; // the pivot's own move is not evidence
                i += 1;
            }
            let (left, right) = std::mem::take(&mut xs).split_at_mut(j + 1);
            was_balanced = left.len().min(right.len()) >= n / 8;
            was_partitioned = !swapped;

            // Recurse into the smaller side, loop on the larger: stack <= lg n.
            // Everything on the right is >= pv, so pv is the right side's pred.
            if left.len() < right.len() {
                self.sort(left, limit, pred);
                (xs, pred) = (right, Some(pv));
            } else {
                self.sort(right, limit, Some(pv));
                xs = left;
            }
        }
    }
}

fn budget(n: usize) -> u32 {
    2 * n.ilog2()
}

/// how often the depth budget ran out, and on how many elements
#[derive(Clone, Copy, Default)]
struct Fallback {
    ranges: usize,
    elems: usize,
}

/// sort a copy with a plain counting comparator; check the result
fn count(input: &[usize], layer: u8) -> (u64, Fallback) {
    let mut xs = input.to_vec();
    let mut cmps = 0u64;
    let mut s = Sorter {
        cmp: |a: usize, b: usize| {
            cmps += 1;
            a.cmp(&b)
        },
        layer,
        fallbacks: 0,
        heap_elems: 0,
    };
    s.sort(&mut xs, budget(input.len()), None);
    let fb = Fallback {
        ranges: s.fallbacks,
        elems: s.heap_elems,
    };
    assert!(xs.is_sorted(), "v{layer} did not sort");
    (cmps, fb)
}

/// McIlroy's adversary (example 12) against one layer: returns the frozen
/// values, an input that makes that layer repeat its work.
fn killer(n: usize, layer: u8) -> Vec<usize> {
    let gas = n;
    let mut val = vec![gas; n];
    let (mut nsolid, mut candidate) = (0, 0);
    let mut idx: Vec<usize> = (0..n).collect();
    let mut s = Sorter {
        // the closure borrows val, nsolid, candidate mutably until s is dropped
        cmp: |x: usize, y: usize| {
            if val[x] == gas && val[y] == gas {
                val[if x == candidate { x } else { y }] = nsolid;
                nsolid += 1;
            }
            if val[x] == gas {
                candidate = x;
            } else if val[y] == gas {
                candidate = y;
            }
            val[x].cmp(&val[y])
        },
        layer,
        fallbacks: 0,
        heap_elems: 0,
    };
    s.sort(&mut idx, budget(n), None);
    val
}

fn main() {
    // n = 50,000: v0 is quadratic on the killer inputs (several hundred
    // million comparisons each), and every killer is generated by one more
    // such run; at 100,000 that work would be four times larger.
    const N: usize = 50_000;
    let mut r = Pcg::new(1, 2);
    let mut random: Vec<usize> = (0..N).collect();
    for i in (1..N).rev() {
        // Fisher-Yates over IntN, as in Go
        let j = r.int_n(i + 1);
        random.swap(i, j);
    }
    let inputs: Vec<(&str, Vec<usize>)> = vec![
        ("random", random),
        ("sorted", (0..N).collect()),
        ("reversed", (0..N).rev().collect()),
        ("all-equal", vec![7; N]),
        ("organ-pipe", (0..N).map(|i| i.min(N - 1 - i)).collect()),
        ("killer of v0", killer(N, 0)),
        ("killer of v1", killer(N, 1)),
        ("killer of v3", killer(N, 3)),
    ];

    // Count everything once: 4 layers + the standard library per input.
    let mut cmps = vec![[0u64; 5]; inputs.len()];
    let mut falls = vec![[Fallback::default(); 4]; inputs.len()];
    for (k, (_, xs)) in inputs.iter().enumerate() {
        for layer in 0..4 {
            (cmps[k][layer], falls[k][layer]) = count(xs, layer as u8);
        }
        let mut c = 0u64;
        xs.clone().sort_unstable_by(|a, b| {
            c += 1;
            a.cmp(b)
        });
        cmps[k][4] = c;
    }

    let nf = N as f64;
    let lg = nf.log2();
    println!(
        "n = {N}, n lg n = {:.0}, depth budget 2*floor(lg n) = {}\n",
        nf * lg,
        budget(N)
    );
    let header = |title: &str| {
        print!(
            "{title:<14} {:>10} {:>10} {:>10} {:>10} {:>13}",
            "v0 quick", "v1 +heap", "v2 +ins", "v3 +pdq", "sort_unstable"
        );
    };
    header("comparisons");
    println!();
    for (k, (name, _)) in inputs.iter().enumerate() {
        print!("{name:<14}");
        for (i, c) in cmps[k].iter().enumerate() {
            let w = if i == 4 { 13 } else { 10 };
            print!(" {c:>w$}");
        }
        println!();
    }

    println!(
        "\n{:<14} {:>21} {:>21} {:>21}",
        "heapsort", "v1 +heap", "v2 +ins", "v3 +pdq"
    );
    println!(
        "{:<14} {:>21} {:>21} {:>21}",
        "fallbacks", "ranges / elements", "ranges / elements", "ranges / elements"
    );
    for (k, (name, _)) in inputs.iter().enumerate() {
        print!("{name:<14}");
        for f in &falls[k][1..] {
            print!(" {:>10} / {:>8}", f.ranges, f.elems);
        }
        println!();
    }

    println!();
    header("/ (n lg n)");
    println!();
    for (k, (name, _)) in inputs.iter().enumerate() {
        print!("{name:<14}");
        for (i, &c) in cmps[k].iter().enumerate() {
            let w = if i == 4 { 13 } else { 10 };
            print!(" {:>w$.2}", c as f64 / (nf * lg));
        }
        println!();
    }
}
```

**Rust output:**

```
n = 50000, n lg n = 780482, depth budget 2*floor(lg n) = 30

comparisons      v0 quick   v1 +heap    v2 +ins    v3 +pdq sort_unstable
random            1172008    1170011     981276     960775        809412
sorted             965531     965531     680795      50002         49999
reversed           990521     990521     705781      50002         49999
all-equal          963323     963323     677739      50002         49999
organ-pipe        3002686    2278754    2268842    2464221        997252
killer of v0    625299992    2891928    2891847    2124361        916115
killer of v1    417584421    2863573    2863492    2848506        875680
killer of v3    313220244    1959373    1816993      75003        814388

heapsort                    v1 +heap               v2 +ins               v3 +pdq
fallbacks          ranges / elements     ranges / elements     ranges / elements
random                192 /      747          5 /       86          0 /        0
sorted                  0 /        0          0 /        0          0 /        0
reversed                0 /        0          0 /        0          0 /        0
all-equal               0 /        0          0 /        0          0 /        0
organ-pipe            496 /    47714        149 /    46382        196 /    44203
killer of v0            2 /    49944          1 /    49942         46 /    32506
killer of v1            2 /    49944          1 /    49942          2 /    49787
killer of v3            1 /    24972          1 /    24972          0 /        0

/ (n lg n)       v0 quick   v1 +heap    v2 +ins    v3 +pdq sort_unstable
random               1.50       1.50       1.26       1.23          1.04
sorted               1.24       1.24       0.87       0.06          0.06
reversed             1.27       1.27       0.90       0.06          0.06
all-equal            1.23       1.23       0.87       0.06          0.06
organ-pipe           3.85       2.92       2.91       3.16          1.28
killer of v0       801.17       3.71       3.71       2.72          1.17
killer of v1       535.03       3.67       3.67       3.65          1.12
killer of v3       401.32       2.51       2.33       0.10          1.04
```

> **Go vs Rust:** Every hand-written column (v0–v3, comparisons and fallbacks) is identical to Go's. The comparator is a generic `FnMut`, so the adversary in `killer` is a closure that mutably borrows its own state for as long as the `Sorter` lives. Hoare's right-hand scan calls `(self.cmp)(xs[j], pv)` rather than `less(pv, xs[j])`: the count is the same, but the adversary's answer depends on argument order, and the swapped call would freeze a different input. Only the last column differs. `sort_unstable_by` (ipnsort) scans the whole slice for one run first (read in the source), so sorted, reversed and all-equal cost exactly n − 1 = 49,999 against Go's 50,011. Organ-pipe costs more in Rust (1.28 n lg n against 0.97), and so does v3's killer (1.04 against 0.16).

**Complexity:** v0 is Θ(n²) on every killer (801 n lg n on its own). From v1 on, every row is Θ(n log n): on the v0 killer v1 makes 3.71 n lg n, after one or two fallbacks that heapsort almost the whole array (49,944 of 50,000 elements). The fallback is not 0 on random input under Musser's every-level budget. v1 fired 192 times there, all on tiny ranges (747 elements in total), insertion sort cut that to 5, and v3 to 0. v3 is Θ(n) on sorted, reversed and all-equal input (50,002 comparisons, as against `slices.SortFunc`'s 50,011). The pdqsort tricks are not free: on organ-pipe v3 makes more comparisons than v2 (3.16 against 2.91 n lg n). Against v3, the adversary backfires the way it did on the standard library in example 12: v3 sorts its own killer in 1.5n comparisons. n = 50,000 rather than 100,000 keeps the quadratic v0 runs to a few seconds.

---

## 14. External sort: runs on disk and a k-way merge

`🔴 hard` · *External merge sort / Θ(n log k)*

When the data does not fit in memory, merge sort still works, because a merge only ever needs the head of each
run. Phase 1 reads M values at a time, sorts them in memory and writes each chunk to its own file as a sorted
run. Phase 2 opens all k runs at once through small read buffers. It keeps one head per run in a min-heap of
(value, run), writes the minimum, and replaces it with the next value from the same run. That is Θ(N log k)
comparisons for the merge, and each phase reads and writes every value exactly once. Replacement selection
changes only phase 1. A heap of M values keeps emitting while new values arrive, and a new value smaller than
the last one written waits for the next run. On random input the runs come out about 2M long, so k is halved.
The program sorts 500,000 int64s (4 MB) with M = 5,000, counts the bytes and the comparisons, checks the result
against an in-memory sort, and deletes its temporary directory at the end.

**Steps:**

1. Write the input to a temporary file. Write a hand-written binary min-heap with a comparison counter, and
   buffered readers and writers that count bytes.
2. Make runs two ways (fixed chunks of M, and replacement selection), then run the same k-way merge on each set
   of runs.
3. Compare the run counts and merge comparisons with N lg k, and confirm that each phase reads and writes every
   value once.

```go
package main

import (
	"bufio"
	"encoding/binary"
	"fmt"
	"io"
	"log"
	"math"
	"math/rand/v2"
	"os"
	"path/filepath"
	"slices"
)

// An external merge sort, small enough to run anywhere but built like the
// real thing: the data lives in files, and only M values are ever held by the
// sort itself. Values are int64, 8 bytes little-endian each.
const (
	N         = 500_000 // values in the input file (4 MB)
	M         = 5_000   // memory budget, in values
	runBufLen = 512     // bytes of read buffer per run during the merge
)

// ioStats counts every byte that crosses the program/file boundary.
type ioStats struct{ read, written int64 }

var stats ioStats

func must(err error) {
	if err != nil {
		log.Fatal(err)
	}
}

// ---- reading and writing int64 files --------------------------------------

type intReader struct {
	f   *os.File
	r   *bufio.Reader
	buf [8]byte
}

func openInts(path string, bufLen int) *intReader {
	f, err := os.Open(path)
	must(err)
	return &intReader{f: f, r: bufio.NewReaderSize(f, bufLen)}
}

// next returns the next value, or ok=false at the end of the file.
func (ir *intReader) next() (v int64, ok bool) {
	_, err := io.ReadFull(ir.r, ir.buf[:])
	if err == io.EOF {
		return 0, false
	}
	must(err) // io.ErrUnexpectedEOF here would mean a torn value
	stats.read += 8
	return int64(binary.LittleEndian.Uint64(ir.buf[:])), true
}

func (ir *intReader) close() { must(ir.f.Close()) }

type intWriter struct {
	f   *os.File
	w   *bufio.Writer
	buf [8]byte
}

func createInts(path string) *intWriter {
	f, err := os.Create(path)
	must(err)
	return &intWriter{f: f, w: bufio.NewWriter(f)} // 4 KiB buffer
}

func (iw *intWriter) put(v int64) {
	binary.LittleEndian.PutUint64(iw.buf[:], uint64(v))
	_, err := iw.w.Write(iw.buf[:])
	must(err)
	stats.written += 8
}

func (iw *intWriter) close() {
	must(iw.w.Flush())
	must(iw.f.Close())
}

// runSet writes runs as numbered files in dir.
type runSet struct {
	dir   string
	paths []string
	lens  []int
	cur   *intWriter
}

func (rs *runSet) start() {
	if rs.cur != nil {
		rs.cur.close()
	}
	p := filepath.Join(rs.dir, fmt.Sprintf("run-%03d", len(rs.paths)))
	rs.paths = append(rs.paths, p)
	rs.lens = append(rs.lens, 0)
	rs.cur = createInts(p)
}

func (rs *runSet) put(v int64) {
	rs.cur.put(v)
	rs.lens[len(rs.lens)-1]++
}

func (rs *runSet) finish() { rs.cur.close() }

// ---- a hand-written binary min-heap ----------------------------------------
// Hand-written (not container/heap) so that the Go and Rust programs count
// exactly the same comparisons. item is (key, tag); the order is given by less.

type item struct {
	key int64
	tag int
}

type minHeap struct {
	a    []item
	less func(x, y item) bool
	cmps int
}

func (h *minHeap) lt(i, j int) bool { h.cmps++; return h.less(h.a[i], h.a[j]) }

func (h *minHeap) siftDown(i int) {
	n := len(h.a)
	for {
		c := 2*i + 1
		if c >= n {
			return
		}
		if c+1 < n && h.lt(c+1, c) {
			c++
		}
		if !h.lt(c, i) {
			return
		}
		h.a[i], h.a[c] = h.a[c], h.a[i]
		i = c
	}
}

func (h *minHeap) push(x item) {
	h.a = append(h.a, x)
	for i := len(h.a) - 1; i > 0; {
		p := (i - 1) / 2
		if !h.lt(i, p) {
			break
		}
		h.a[i], h.a[p] = h.a[p], h.a[i]
		i = p
	}
}

func (h *minHeap) pop() item {
	top := h.a[0]
	last := len(h.a) - 1
	h.a[0] = h.a[last]
	h.a = h.a[:last]
	h.siftDown(0)
	return top
}

// replaceTop swaps the minimum for x: one sift-down instead of pop + push.
func (h *minHeap) replaceTop(x item) { h.a[0] = x; h.siftDown(0) }

// ---- phase 1, two ways ------------------------------------------------------

// fixedRuns: read M values, sort them in memory, write them out as a run.
func fixedRuns(input, dir string) *runSet {
	rs := &runSet{dir: dir}
	in := openInts(input, 4096)
	defer in.close()
	chunk := make([]int64, 0, M) // the whole memory budget
	flush := func() {
		slices.Sort(chunk)
		rs.start()
		for _, v := range chunk {
			rs.put(v)
		}
		chunk = chunk[:0]
	}
	for v, ok := in.next(); ok; v, ok = in.next() {
		chunk = append(chunk, v)
		if len(chunk) == M {
			flush()
		}
	}
	if len(chunk) > 0 {
		flush()
	}
	rs.finish()
	return rs
}

// replacementSelection: a heap of M (run, value) pairs. Pop the smallest and
// write it to the current run; read the next input value into its place. If it
// is >= the value just written it can still join this run, otherwise it is
// tagged for the next run. A run ends when the heap's minimum is tagged for
// the next run. On random input the runs come out about 2M long.
func replacementSelection(input, dir string) *runSet {
	rs := &runSet{dir: dir}
	in := openInts(input, 4096)
	defer in.close()
	h := &minHeap{less: func(x, y item) bool {
		if x.tag != y.tag {
			return x.tag < y.tag
		}
		return x.key < y.key
	}}
	for len(h.a) < M {
		v, ok := in.next()
		if !ok {
			break
		}
		h.a = append(h.a, item{v, 0})
	}
	for i := len(h.a)/2 - 1; i >= 0; i-- {
		h.siftDown(i)
	}
	run := -1
	for len(h.a) > 0 {
		top := h.a[0]
		if top.tag != run {
			run = top.tag
			rs.start()
		}
		rs.put(top.key)
		if v, ok := in.next(); ok {
			tag := run
			if v < top.key {
				tag = run + 1 // too small for the current run
			}
			h.replaceTop(item{v, tag})
		} else {
			h.pop()
		}
	}
	rs.finish()
	return rs
}

// ---- phase 2: the k-way merge ----------------------------------------------

// merge reads all runs at once through small buffers, keeps one head per run
// in a min-heap of (value, run) and writes the minimum each step. Each step is
// one replaceTop: a sift-down of up to lg k levels at up to 2 comparisons per
// level (pick the smaller child, then test it against the parent). A loser
// tree would make about lg k per step.
func merge(rs *runSet, output string) (cmps int) {
	readers := make([]*intReader, len(rs.paths))
	h := &minHeap{less: func(x, y item) bool {
		if x.key != y.key {
			return x.key < y.key
		}
		return x.tag < y.tag // ties: lower run first, so the merge is deterministic
	}}
	for i, p := range rs.paths {
		readers[i] = openInts(p, runBufLen)
		if v, ok := readers[i].next(); ok {
			h.a = append(h.a, item{v, i})
		}
	}
	for i := len(h.a)/2 - 1; i >= 0; i-- {
		h.siftDown(i)
	}
	out := createInts(output)
	for len(h.a) > 0 {
		top := h.a[0]
		out.put(top.key)
		if v, ok := readers[top.tag].next(); ok {
			h.replaceTop(item{v, top.tag})
		} else {
			h.pop()
		}
	}
	out.close()
	for _, r := range readers {
		r.close()
	}
	return h.cmps
}

// verify: the output file holds exactly the in-memory sort of the input.
// Its reads are checking, not sorting, so they are left out of the I/O count.
func verify(output string, want []int64) bool {
	saved := stats
	defer func() { stats = saved }()
	in := openInts(output, 4096)
	defer in.close()
	i := 0
	for v, ok := in.next(); ok; v, ok = in.next() {
		if i >= len(want) || v != want[i] {
			return false
		}
		i++
	}
	return i == len(want)
}

func phase(name string, f func()) {
	before := stats
	f()
	fmt.Printf("  %-34s read %8d B, wrote %8d B\n", name, stats.read-before.read, stats.written-before.written)
}

func main() {
	dir, err := os.MkdirTemp("", "extsort-*")
	must(err)
	defer os.RemoveAll(dir) // every file this program made goes with it

	// The input: N seeded random int64s, written to disk.
	r := rand.New(rand.NewPCG(1, 2))
	values := make([]int64, N) // kept only to verify the result at the end
	input := filepath.Join(dir, "input")
	w := createInts(input)
	for i := range values {
		values[i] = int64(r.Uint64())
		w.put(values[i])
	}
	w.close()
	stats = ioStats{} // count the sort's I/O, not the setup's
	want := slices.Clone(values)
	slices.Sort(want)
	fmt.Printf("N = %d int64 values (%d bytes on disk), memory budget M = %d values\n", N, 8*N, M)

	type result struct {
		runs int
		cmps int
	}
	sortWith := func(title string, makeRuns func(string, string) *runSet) result {
		fmt.Printf("\n%s\n", title)
		var rs *runSet
		phase("phase 1, make runs:", func() { rs = makeRuns(input, dir) })
		k := len(rs.paths)
		fmt.Printf("  runs: %d, average length %.1f = %.2f M (shortest %d, longest %d)\n",
			k, float64(N)/float64(k), float64(N)/float64(k)/M, slices.Min(rs.lens), slices.Max(rs.lens))
		output := filepath.Join(dir, "output")
		var cmps int
		phase(fmt.Sprintf("phase 2, %d-way merge:", k), func() { cmps = merge(rs, output) })
		nlgk := float64(N) * math.Log2(float64(k))
		fmt.Printf("  merge comparisons: %d = %.2f x N lg k (N lg k = %.0f)\n", cmps, float64(cmps)/nlgk, nlgk)
		fmt.Printf("  check: %d values, equal to an in-memory sort: %v\n", N, verify(output, want))
		for _, p := range rs.paths { // the runs are no longer needed
			must(os.Remove(p))
		}
		return result{k, cmps}
	}

	fixed := sortWith("1) fixed runs: read M values, sort them, write a run; then merge all runs", fixedRuns)
	repl := sortWith("2) replacement selection: a heap of M emits while input arrives; then merge", replacementSelection)

	fmt.Printf("\nreplacement selection: %d runs instead of %d (%.2fx fewer), merge comparisons %.1f%% fewer\n",
		repl.runs, fixed.runs, float64(fixed.runs)/float64(repl.runs),
		100*(1-float64(repl.cmps)/float64(fixed.cmps)))
	fmt.Printf("I/O per sort: every value read %.0f times and written %.0f times\n",
		float64(stats.read)/(2*8*N), float64(stats.written)/(2*8*N))
	fmt.Printf("read buffers in the merge: %d runs x %d B, about %d values per run\n", fixed.runs, runBufLen, runBufLen/8)
}
```

**Output:**

```
N = 500000 int64 values (4000000 bytes on disk), memory budget M = 5000 values

1) fixed runs: read M values, sort them, write a run; then merge all runs
  phase 1, make runs:                read  4000000 B, wrote  4000000 B
  runs: 100, average length 5000.0 = 1.00 M (shortest 5000, longest 5000)
  phase 2, 100-way merge:            read  4000000 B, wrote  4000000 B
  merge comparisons: 5290029 = 1.59 x N lg k (N lg k = 3321928)
  check: 500000 values, equal to an in-memory sort: true

2) replacement selection: a heap of M emits while input arrives; then merge
  phase 1, make runs:                read  4000000 B, wrote  4000000 B
  runs: 51, average length 9803.9 = 1.96 M (shortest 1511, longest 10133)
  phase 2, 51-way merge:             read  4000000 B, wrote  4000000 B
  merge comparisons: 4353814 = 1.54 x N lg k (N lg k = 2836213)
  check: 500000 values, equal to an in-memory sort: true

replacement selection: 51 runs instead of 100 (1.96x fewer), merge comparisons 17.7% fewer
I/O per sort: every value read 2 times and written 2 times
read buffers in the merge: 100 runs x 512 B, about 64 values per run
```

**Rust:**

```rust
use std::cell::Cell;
use std::fs::{self, File};
use std::io::{self, BufRead, BufReader, BufWriter, Read, Write};
use std::path::{Path, PathBuf};

// An external merge sort, small enough to run anywhere but built like the
// real thing: the data lives in files, and only M values are ever held by the
// sort itself. Values are i64, 8 bytes little-endian each.
const N: usize = 500_000; // values in the input file (4 MB)
const M: usize = 5_000; // memory budget, in values
const RUN_BUF_LEN: usize = 512; // bytes of read buffer per run during the merge

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM), so the input file holds
// the same 500,000 values as the Go program's.
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

/// Counts every byte that crosses the program/file boundary. No mutable
/// globals in Rust: every reader and writer holds a shared &Stats, and the
/// Cells let them all bump the same counters.
#[derive(Default)]
struct Stats {
    read: Cell<u64>,
    written: Cell<u64>,
}

/// The temporary directory, removed with everything in it when this guard is
/// dropped: at the end of main, on an early `?` return, or during a panic.
struct TempDir(PathBuf);

impl Drop for TempDir {
    fn drop(&mut self) {
        let _ = fs::remove_dir_all(&self.0);
    }
}

// ---- reading and writing i64 files -----------------------------------------

struct IntReader<'s> {
    r: BufReader<File>,
    stats: &'s Stats,
}

impl<'s> IntReader<'s> {
    fn open(path: &Path, buf_len: usize, stats: &'s Stats) -> io::Result<Self> {
        Ok(IntReader {
            r: BufReader::with_capacity(buf_len, File::open(path)?),
            stats,
        })
    }

    /// the next value, or None at the end of the file. read_exact reports a
    /// clean end and a torn value the same way, so look at the buffer first.
    fn next(&mut self) -> io::Result<Option<i64>> {
        if self.r.fill_buf()?.is_empty() {
            return Ok(None);
        }
        let mut b = [0u8; 8];
        self.r.read_exact(&mut b)?;
        self.stats.read.set(self.stats.read.get() + 8);
        Ok(Some(i64::from_le_bytes(b)))
    }
}

struct IntWriter<'s> {
    w: BufWriter<File>,
    stats: &'s Stats,
}

impl<'s> IntWriter<'s> {
    fn create(path: &Path, stats: &'s Stats) -> io::Result<Self> {
        // 4 KiB like Go's bufio default (Rust's own default is 8 KiB)
        Ok(IntWriter {
            w: BufWriter::with_capacity(4096, File::create(path)?),
            stats,
        })
    }

    fn put(&mut self, v: i64) -> io::Result<()> {
        self.w.write_all(&v.to_le_bytes())?;
        self.stats.written.set(self.stats.written.get() + 8);
        Ok(())
    }

    /// flush explicitly: BufWriter's Drop flushes too, but swallows the error
    fn close(mut self) -> io::Result<()> {
        self.w.flush()
    }
}

/// runs written as numbered files in dir
struct RunSet<'s> {
    dir: PathBuf,
    paths: Vec<PathBuf>,
    lens: Vec<usize>,
    cur: Option<IntWriter<'s>>,
    stats: &'s Stats,
}

impl<'s> RunSet<'s> {
    fn new(dir: &Path, stats: &'s Stats) -> Self {
        RunSet {
            dir: dir.to_path_buf(),
            paths: Vec::new(),
            lens: Vec::new(),
            cur: None,
            stats,
        }
    }

    fn start(&mut self) -> io::Result<()> {
        if let Some(w) = self.cur.take() {
            w.close()?;
        }
        let p = self.dir.join(format!("run-{:03}", self.paths.len()));
        self.cur = Some(IntWriter::create(&p, self.stats)?);
        self.paths.push(p);
        self.lens.push(0);
        Ok(())
    }

    fn put(&mut self, v: i64) -> io::Result<()> {
        self.cur.as_mut().expect("start() first").put(v)?;
        *self.lens.last_mut().expect("start() first") += 1;
        Ok(())
    }

    fn finish(&mut self) -> io::Result<()> {
        self.cur.take().map_or(Ok(()), IntWriter::close)
    }
}

// ---- a hand-written binary min-heap -----------------------------------------
// Hand-written (not BinaryHeap) so that the Go and Rust programs count exactly
// the same comparisons. Item is (key, tag); the order is given by `less`.

#[derive(Clone, Copy)]
struct Item {
    key: i64,
    tag: usize,
}

struct MinHeap<F: Fn(&Item, &Item) -> bool> {
    a: Vec<Item>,
    less: F,
    cmps: u64,
}

impl<F: Fn(&Item, &Item) -> bool> MinHeap<F> {
    fn new(less: F, cap: usize) -> Self {
        MinHeap {
            a: Vec::with_capacity(cap),
            less,
            cmps: 0,
        }
    }

    fn lt(&mut self, i: usize, j: usize) -> bool {
        self.cmps += 1;
        (self.less)(&self.a[i], &self.a[j])
    }

    fn sift_down(&mut self, mut i: usize) {
        let n = self.a.len();
        loop {
            let mut c = 2 * i + 1;
            if c >= n {
                return;
            }
            if c + 1 < n && self.lt(c + 1, c) {
                c += 1;
            }
            if !self.lt(c, i) {
                return;
            }
            self.a.swap(i, c);
            i = c;
        }
    }

    fn heapify(&mut self) {
        for i in (0..self.a.len() / 2).rev() {
            self.sift_down(i);
        }
    }

    fn pop(&mut self) -> Item {
        let top = self.a.swap_remove(0); // the last item moves to the root
        if !self.a.is_empty() {
            self.sift_down(0);
        }
        top
    }

    /// swap the minimum for x: one sift-down instead of pop + push
    fn replace_top(&mut self, x: Item) {
        self.a[0] = x;
        self.sift_down(0);
    }
}

// ---- phase 1, two ways --------------------------------------------------------

/// read M values, sort them in memory, write them out as a run
fn fixed_runs<'s>(input: &Path, dir: &Path, stats: &'s Stats) -> io::Result<RunSet<'s>> {
    let mut rs = RunSet::new(dir, stats);
    let mut inp = IntReader::open(input, 4096, stats)?;
    let mut chunk: Vec<i64> = Vec::with_capacity(M); // the whole memory budget
    // a nested fn, not a closure: it captures nothing, and taking rs as a
    // parameter avoids holding a borrow of it across the loop
    fn flush(chunk: &mut Vec<i64>, rs: &mut RunSet) -> io::Result<()> {
        chunk.sort_unstable();
        rs.start()?;
        for &v in chunk.iter() {
            rs.put(v)?;
        }
        chunk.clear();
        Ok(())
    }
    while let Some(v) = inp.next()? {
        chunk.push(v);
        if chunk.len() == M {
            flush(&mut chunk, &mut rs)?;
        }
    }
    if !chunk.is_empty() {
        flush(&mut chunk, &mut rs)?;
    }
    rs.finish()?;
    Ok(rs)
}

/// A heap of M (run, value) pairs. Pop the smallest and write it to the
/// current run; read the next input value into its place. If it is >= the
/// value just written it can still join this run, otherwise it is tagged for
/// the next run. A run ends when the heap's minimum is tagged for the next
/// run. On random input the runs come out about 2M long.
fn replacement_selection<'s>(input: &Path, dir: &Path, stats: &'s Stats) -> io::Result<RunSet<'s>> {
    let mut rs = RunSet::new(dir, stats);
    let mut inp = IntReader::open(input, 4096, stats)?;
    // (tag, key) order: tuples compare lexicographically
    let mut h = MinHeap::new(|x: &Item, y: &Item| (x.tag, x.key) < (y.tag, y.key), M);
    while h.a.len() < M {
        match inp.next()? {
            Some(v) => h.a.push(Item { key: v, tag: 0 }),
            None => break,
        }
    }
    h.heapify();
    let mut run = None;
    while let Some(&top) = h.a.first() {
        if run != Some(top.tag) {
            run = Some(top.tag);
            rs.start()?;
        }
        rs.put(top.key)?;
        match inp.next()? {
            Some(v) => {
                // too small for the current run: it waits for the next one
                let tag = if v < top.key { top.tag + 1 } else { top.tag };
                h.replace_top(Item { key: v, tag });
            }
            None => {
                h.pop();
            }
        }
    }
    rs.finish()?;
    Ok(rs)
}

// ---- phase 2: the k-way merge -------------------------------------------------

/// Read all runs at once through small buffers, keep one head per run in a
/// min-heap of (value, run) and write the minimum each step. Each step is one
/// replace_top: a sift-down of up to lg k levels at up to 2 comparisons per
/// level. A loser tree would make about lg k per step.
fn merge(rs: &RunSet, output: &Path, stats: &Stats) -> io::Result<u64> {
    let mut readers = Vec::with_capacity(rs.paths.len());
    // ties: lower run first, so the merge is deterministic
    let mut h = MinHeap::new(
        |x: &Item, y: &Item| (x.key, x.tag) < (y.key, y.tag),
        rs.paths.len(),
    );
    for (i, p) in rs.paths.iter().enumerate() {
        let mut r = IntReader::open(p, RUN_BUF_LEN, stats)?;
        if let Some(v) = r.next()? {
            h.a.push(Item { key: v, tag: i });
        }
        readers.push(r);
    }
    h.heapify();
    let mut out = IntWriter::create(output, stats)?;
    while let Some(&top) = h.a.first() {
        out.put(top.key)?;
        match readers[top.tag].next()? {
            Some(v) => h.replace_top(Item {
                key: v,
                tag: top.tag,
            }),
            None => {
                h.pop();
            }
        }
    }
    out.close()?;
    Ok(h.cmps) // the readers' files close when `readers` is dropped
}

/// the output file holds exactly the in-memory sort of the input. It counts
/// into its own throwaway Stats: checking is not part of the sort's I/O.
fn verify(output: &Path, want: &[i64]) -> io::Result<bool> {
    let scratch = Stats::default();
    let mut inp = IntReader::open(output, 4096, &scratch)?;
    let mut i = 0;
    while let Some(v) = inp.next()? {
        if want.get(i) != Some(&v) {
            return Ok(false);
        }
        i += 1;
    }
    Ok(i == want.len())
}

fn phase<T>(name: &str, stats: &Stats, f: impl FnOnce() -> io::Result<T>) -> io::Result<T> {
    let (r0, w0) = (stats.read.get(), stats.written.get());
    let out = f()?;
    println!(
        "  {name:<34} read {:>8} B, wrote {:>8} B",
        stats.read.get() - r0,
        stats.written.get() - w0
    );
    Ok(out)
}

type MakeRuns = for<'s> fn(&Path, &Path, &'s Stats) -> io::Result<RunSet<'s>>;

fn main() -> io::Result<()> {
    let dir = TempDir(std::env::temp_dir().join(format!("extsort-{}", std::process::id())));
    fs::create_dir(&dir.0)?;

    // The input: N seeded random i64s, written to disk.
    let mut r = Pcg::new(1, 2);
    let values: Vec<i64> = (0..N).map(|_| r.next_u64() as i64).collect(); // kept only to verify
    let input = dir.0.join("input");
    let setup = Stats::default(); // the setup's I/O is not the sort's
    let mut w = IntWriter::create(&input, &setup)?;
    for &v in &values {
        w.put(v)?;
    }
    w.close()?;
    let mut want = values.clone();
    want.sort_unstable();
    println!(
        "N = {N} int64 values ({} bytes on disk), memory budget M = {M} values",
        8 * N
    );

    let stats = Stats::default();
    let sort_with = |title: &str, make_runs: MakeRuns| -> io::Result<(usize, u64)> {
        println!("\n{title}");
        let rs = phase("phase 1, make runs:", &stats, || {
            make_runs(&input, &dir.0, &stats)
        })?;
        let k = rs.paths.len();
        let avg = N as f64 / k as f64;
        println!(
            "  runs: {k}, average length {avg:.1} = {:.2} M (shortest {}, longest {})",
            avg / M as f64,
            rs.lens.iter().min().unwrap(),
            rs.lens.iter().max().unwrap()
        );
        let output = dir.0.join("output");
        let cmps = phase(&format!("phase 2, {k}-way merge:"), &stats, || {
            merge(&rs, &output, &stats)
        })?;
        let nlgk = N as f64 * (k as f64).log2();
        println!(
            "  merge comparisons: {cmps} = {:.2} x N lg k (N lg k = {nlgk:.0})",
            cmps as f64 / nlgk
        );
        println!(
            "  check: {N} values, equal to an in-memory sort: {}",
            verify(&output, &want)?
        );
        for p in &rs.paths {
            fs::remove_file(p)?; // the runs are no longer needed
        }
        Ok((k, cmps))
    };

    let (fixed_k, fixed_c) = sort_with(
        "1) fixed runs: read M values, sort them, write a run; then merge all runs",
        fixed_runs,
    )?;
    let (repl_k, repl_c) = sort_with(
        "2) replacement selection: a heap of M emits while input arrives; then merge",
        replacement_selection,
    )?;

    println!(
        "\nreplacement selection: {repl_k} runs instead of {fixed_k} ({:.2}x fewer), merge comparisons {:.1}% fewer",
        fixed_k as f64 / repl_k as f64,
        100.0 * (1.0 - repl_c as f64 / fixed_c as f64)
    );
    println!(
        "I/O per sort: every value read {:.0} times and written {:.0} times",
        stats.read.get() as f64 / (2 * 8 * N) as f64,
        stats.written.get() as f64 / (2 * 8 * N) as f64
    );
    println!(
        "read buffers in the merge: {fixed_k} runs x {RUN_BUF_LEN} B, about {} values per run",
        RUN_BUF_LEN / 8
    );
    Ok(()) // `dir` drops here and removes the temporary directory
}
```

**Rust output:**

```
N = 500000 int64 values (4000000 bytes on disk), memory budget M = 5000 values

1) fixed runs: read M values, sort them, write a run; then merge all runs
  phase 1, make runs:                read  4000000 B, wrote  4000000 B
  runs: 100, average length 5000.0 = 1.00 M (shortest 5000, longest 5000)
  phase 2, 100-way merge:            read  4000000 B, wrote  4000000 B
  merge comparisons: 5290029 = 1.59 x N lg k (N lg k = 3321928)
  check: 500000 values, equal to an in-memory sort: true

2) replacement selection: a heap of M emits while input arrives; then merge
  phase 1, make runs:                read  4000000 B, wrote  4000000 B
  runs: 51, average length 9803.9 = 1.96 M (shortest 1511, longest 10133)
  phase 2, 51-way merge:             read  4000000 B, wrote  4000000 B
  merge comparisons: 4353814 = 1.54 x N lg k (N lg k = 2836213)
  check: 500000 values, equal to an in-memory sort: true

replacement selection: 51 runs instead of 100 (1.96x fewer), merge comparisons 17.7% fewer
I/O per sort: every value read 2 times and written 2 times
read buffers in the merge: 100 runs x 512 B, about 64 values per run
```

> **Go vs Rust:** Identical output, byte for byte. Both programs hand-write the heap because the standard ones count differently: Rust's `BinaryHeap::pop` sifts the hole all the way to the bottom and then back up (`sift_down_to_bottom`, read in the source, not run). Go counts bytes in a package-level `stats` variable. Safe Rust has no mutable globals, so every reader and writer holds one shared `&Stats` of `Cell`s instead. Cleanup is a `Drop` guard on the temporary directory where Go has `defer os.RemoveAll`, so it also runs on an early `?` return. `read_exact` reports a clean end of file and a torn value the same way, so `next` checks `fill_buf` first. Go tells the two apart with `io.EOF` against `io.ErrUnexpectedEOF`.

**Complexity:** phase 1 costs Θ(N log M) comparisons in memory, and the merge Θ(N log k). The 100-way merge made 5,290,029 comparisons, 1.59 × N lg k, because a binary heap's sift-down spends up to 2 comparisons per level. Memory is Θ(M) plus one small buffer per run. Each phase read and wrote all 4,000,000 bytes exactly once. Replacement selection produced 51 runs of 1.96 M on average instead of 100 runs of M, and cut the merge comparisons by 17.7%. N is 500,000 rather than exercise 14's 10⁶ so the temporary files stay at a few MB. The ratio N/M = 100 runs is the same.

---

## 15. Comparisons are not time: the full race

`🔴 hard` · *Measured comparison*

The comparison count is the textbook cost model: it is exact, the same on every machine, and the n lg n lower
bound is stated in it. It is not the clock. This program races every sort in the lesson at n = 1,000,000 on five
input shapes, and counts first and times second. The counted table needs no caveats. Top-down merge sort makes
the fewest comparisons on random input, about 1% above the lg(n!) floor. Heapsort makes almost twice as many,
because its sift-down spends two comparisons per level: one to pick the larger child, one against the parent.
The random-pivot quicksort makes 1.5 times as many. The timed table then shows that a comparison has no fixed
price. A top-down merge writes every element into the buffer and back again on every level, while a Hoare
partition swaps only the misplaced elements, about n/6 per level. Every sort here is Θ(n log n) on these shapes, and the clock still ranks them differently.

**Steps:**

1. Build the six hand-written sorts from this lesson (top-down and bottom-up merge, random-pivot Hoare quicksort,
   3-way quicksort, heapsort, introsort), each taking a `cmp func(a, b int) int`, as `slices.SortFunc` does.
2. Count comparator calls on random, sorted, reversed, few-distinct (k = 10) and organ-pipe input. Then time
   each sort with a plain `cmp.Compare`, copying the pristine input inside every timed run, subtracting a
   copy-only control, and keeping the best of 3 interleaved rounds.
3. Read the last column, ns per comparison on random input, and the computed lines under the table. They show
   where the comparison ranking and the time ranking disagree.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
	"time"
)

// Every sort below takes its comparator as a func value, exactly as
// slices.SortFunc does, so the race is fair: one indirect call per comparison
// for every comparator-based row. Table 1 counts those calls; table 2 times
// the same sorts with a plain, non-counting cmp.Compare.
type cmpFn = func(a, b int) int

// ---- top-down merge sort: one scratch buffer, copy back after each merge ----

func mergeTopDown(xs []int, c cmpFn) { mergeRec(xs, make([]int, len(xs)), c) }

func mergeRec(xs, buf []int, c cmpFn) {
	n := len(xs)
	if n < 2 {
		return
	}
	mid := n / 2
	mergeRec(xs[:mid], buf[:mid], c)
	mergeRec(xs[mid:], buf[mid:], c)
	merge(xs[:mid], xs[mid:], buf[:n], c)
	copy(xs, buf[:n])
}

// merge merges the sorted runs a and b into out. Ties take a: stable.
func merge(a, b, out []int, c cmpFn) {
	i, j, k := 0, 0, 0
	for i < len(a) && j < len(b) {
		if c(b[j], a[i]) < 0 {
			out[k] = b[j]
			j++
		} else {
			out[k] = a[i]
			i++
		}
		k++
	}
	k += copy(out[k:], a[i:])
	copy(out[k:], b[j:])
}

// ---- bottom-up merge sort: widths 1, 2, 4, ...; the buffers swap roles ----

func mergeBottomUp(xs []int, c cmpFn) {
	n := len(xs)
	src, dst := xs, make([]int, n)
	inBuf := false // after an odd number of passes the data sits in the buffer
	for w := 1; w < n; w *= 2 {
		for lo := 0; lo < n; lo += 2 * w {
			mid, hi := min(lo+w, n), min(lo+2*w, n)
			merge(src[lo:mid], src[mid:hi], dst[lo:hi], c)
		}
		src, dst = dst, src
		inBuf = !inBuf
	}
	if inBuf {
		copy(xs, src)
	}
}

// ---- Hoare partition (CLRS), pivot = xs[0]; returns j: xs[:j+1] <= p <= xs[j+1:] ----

func hoare(xs []int, c cmpFn) int {
	p := xs[0]
	i, j := 0, len(xs)
	for {
		j--
		for c(xs[j], p) > 0 {
			j--
		}
		for c(xs[i], p) < 0 {
			i++
		}
		if i >= j {
			return j
		}
		xs[i], xs[j] = xs[j], xs[i]
		i++
	}
}

// ---- quicksort, random pivot: safe on every shape (in expectation) ----

func quickRandom(xs []int, c cmpFn, r *rand.Rand) {
	for len(xs) > 1 {
		p := r.IntN(len(xs))
		xs[0], xs[p] = xs[p], xs[0]
		s := hoare(xs, c)
		left, right := xs[:s+1], xs[s+1:]
		if len(left) < len(right) { // recurse on the smaller side, loop on the larger
			quickRandom(left, c, r)
			xs = right
		} else {
			quickRandom(right, c, r)
			xs = left
		}
	}
}

// ---- 3-way quicksort (Dijkstra), random pivot: one cmp call per element per pass ----

func quick3(xs []int, c cmpFn, r *rand.Rand) {
	for len(xs) > 1 {
		p := xs[r.IntN(len(xs))]
		lt, i, gt := 0, 0, len(xs) // xs[:lt] < p, xs[lt:i] == p, xs[gt:] > p
		for i < gt {
			switch d := c(xs[i], p); {
			case d < 0:
				xs[lt], xs[i] = xs[i], xs[lt]
				lt++
				i++
			case d > 0:
				gt--
				xs[i], xs[gt] = xs[gt], xs[i]
			default:
				i++
			}
		}
		left, right := xs[:lt], xs[gt:] // the == p block is finished
		if len(left) < len(right) {
			quick3(left, c, r)
			xs = right
		} else {
			quick3(right, c, r)
			xs = left
		}
	}
}

// ---- heapsort in place (sift-down as in Go's siftDownOrdered) ----

func heapSort(xs []int, c cmpFn) {
	n := len(xs)
	for i := n/2 - 1; i >= 0; i-- {
		siftDown(xs, i, n, c)
	}
	for end := n - 1; end > 0; end-- {
		xs[0], xs[end] = xs[end], xs[0]
		siftDown(xs, 0, end, c)
	}
}

func siftDown(xs []int, root, n int, c cmpFn) {
	for {
		child := 2*root + 1 // the jump from i to 2i+1: a cache miss on big arrays
		if child >= n {
			return
		}
		if child+1 < n && c(xs[child], xs[child+1]) < 0 {
			child++
		}
		if c(xs[root], xs[child]) >= 0 {
			return
		}
		xs[root], xs[child] = xs[child], xs[root]
		root = child
	}
}

// ---- introsort: median-of-3 Hoare quicksort, depth 2*floor(lg n) -> heapsort, insertion <= 12 ----

func introsort(xs []int, c cmpFn) { introRec(xs, c, 2*(bits.Len(uint(len(xs)))-1)) }

func introRec(xs []int, c cmpFn, depth int) {
	for len(xs) > 12 {
		if depth == 0 {
			heapSort(xs, c) // the guarantee: Θ(n log n) whatever the pivots did
			return
		}
		depth--
		m := median3(xs, 0, len(xs)/2, len(xs)-1, c)
		xs[0], xs[m] = xs[m], xs[0]
		s := hoare(xs, c)
		left, right := xs[:s+1], xs[s+1:]
		if len(left) < len(right) {
			introRec(left, c, depth)
			xs = right
		} else {
			introRec(right, c, depth)
			xs = left
		}
	}
	for i := 1; i < len(xs); i++ { // insertion sort finishes the small range
		for j := i; j > 0 && c(xs[j], xs[j-1]) < 0; j-- {
			xs[j], xs[j-1] = xs[j-1], xs[j]
		}
	}
}

func median3(xs []int, a, b, m int, c cmpFn) int {
	if c(xs[b], xs[a]) < 0 {
		a, b = b, a // now xs[a] <= xs[b]
	}
	if c(xs[m], xs[b]) < 0 {
		if c(xs[m], xs[a]) < 0 {
			return a
		}
		return m
	}
	return b
}

// ---- the race ----

type algo struct {
	name string
	sort func([]int, cmpFn) // nil: slices.Sort, which takes no comparator
}

var algos = []algo{
	{"merge top-down", mergeTopDown},
	{"merge bottom-up", mergeBottomUp},
	{"quick random", func(xs []int, c cmpFn) { quickRandom(xs, c, rand.New(rand.NewPCG(3, 4))) }},
	{"quick 3-way", func(xs []int, c cmpFn) { quick3(xs, c, rand.New(rand.NewPCG(3, 4))) }},
	{"heapsort", heapSort},
	{"introsort", introsort},
	{"slices.SortFunc", func(xs []int, c cmpFn) { slices.SortFunc(xs, c) }},
	{"slices.SortStableFunc", func(xs []int, c cmpFn) { slices.SortStableFunc(xs, c) }},
	{"slices.Sort", nil},
}

// msOnce copies the pristine input into work INSIDE the timed run (sorting
// sorted data would measure nothing) and returns milliseconds.
func msOnce(work, pristine []int, f func([]int)) float64 {
	start := time.Now()
	copy(work, pristine)
	f(work)
	return float64(time.Since(start).Nanoseconds()) / 1e6
}

func main() {
	const n = 1_000_000
	r := rand.New(rand.NewPCG(1, 2))
	random := make([]int, n)
	for i := range random {
		random[i] = i
	}
	for i := n - 1; i > 0; i-- { // Fisher-Yates over IntN
		j := r.IntN(i + 1)
		random[i], random[j] = random[j], random[i]
	}
	sorted, reversed, few, organ := make([]int, n), make([]int, n), make([]int, n), make([]int, n)
	for i := range n {
		sorted[i], reversed[i], few[i] = i, n-1-i, r.IntN(10)
		organ[i] = min(i, n-1-i) // 0, 1, ..., n/2-1, n/2-1, ..., 1, 0
	}
	shapes := []struct {
		name string
		xs   []int
	}{{"random", random}, {"sorted", sorted}, {"reversed", reversed}, {"few k=10", few}, {"organ-pipe", organ}}

	fmt.Printf("n = %d; comparator sorts take cmp func(a, b int) int; one call = one comparison\n\n", n)
	fmt.Println("1. Comparisons (deterministic)")
	fmt.Printf("%-22s", "algorithm")
	for _, s := range shapes {
		fmt.Printf(" %11s", s.name)
	}
	fmt.Println()
	work, want := make([]int, n), make([]int, n)
	cmps := map[string][]int{}
	allOK := true
	for _, a := range algos[:len(algos)-1] {
		fmt.Printf("%-22s", a.name)
		for _, s := range shapes {
			copy(want, s.xs)
			slices.Sort(want)
			copy(work, s.xs)
			count := 0
			a.sort(work, func(x, y int) int { count++; return cmp.Compare(x, y) })
			allOK = allOK && slices.Equal(work, want)
			cmps[a.name] = append(cmps[a.name], count)
			fmt.Printf(" %11d", count)
		}
		fmt.Println()
	}
	lgn := math.Log2(n)
	fmt.Printf("%-22s %11.0f   (the lg(n!) floor is about n lg n - 1.44n = %.0f)\n",
		"n lg n", n*lgn, n*lgn-math.Log2E*n)
	fmt.Println("every sort's output equals slices.Sort's on every shape:", allOK)

	// Best of 3 ROUNDS, each round timing every sort on every shape once: a
	// burst of load on a shared machine then spoils one round of several
	// sorts, not all three tries of one sort.
	ms := map[string][]float64{}
	ctrl := slices.Repeat([]float64{math.Inf(1)}, len(shapes))
	for _, a := range algos {
		ms[a.name] = slices.Repeat([]float64{math.Inf(1)}, len(shapes))
	}
	for range 3 {
		for si, s := range shapes {
			ctrl[si] = min(ctrl[si], msOnce(work, s.xs, func([]int) {})) // copy-only control
			for _, a := range algos {
				f := slices.Sort[[]int]
				if a.sort != nil {
					f = func(xs []int) { a.sort(xs, cmp.Compare[int]) }
				}
				ms[a.name][si] = min(ms[a.name][si], msOnce(work, s.xs, f))
			}
		}
	}
	fmt.Println("\n2. Milliseconds per sort (copy-only control subtracted; best of 3 rounds)")
	fmt.Printf("%-22s", "algorithm")
	for _, s := range shapes {
		fmt.Printf(" %11s", s.name)
	}
	fmt.Printf(" %13s\n", "ns/cmp random")
	for _, a := range algos {
		fmt.Printf("%-22s", a.name)
		for si := range shapes {
			ms[a.name][si] = max(ms[a.name][si]-ctrl[si], 0)
			fmt.Printf(" %11.1f", ms[a.name][si])
		}
		if c := cmps[a.name]; c != nil {
			fmt.Printf(" %13.2f", ms[a.name][0]*1e6/float64(c[0]))
		}
		fmt.Println()
	}

	// What the two tables say, computed from this run's numbers.
	fewest, fastest, most, slowest, cheap, dear := "", "", "", "", "", ""
	nsPerCmp := func(a string) float64 { return ms[a][0] * 1e6 / float64(cmps[a][0]) }
	for _, a := range algos[:len(algos)-1] {
		first := fewest == ""
		if first || cmps[a.name][0] < cmps[fewest][0] {
			fewest = a.name
		}
		if first || cmps[a.name][0] > cmps[most][0] {
			most = a.name
		}
		if first || ms[a.name][0] < ms[fastest][0] {
			fastest = a.name
		}
		if first || ms[a.name][0] > ms[slowest][0] {
			slowest = a.name
		}
		if first || nsPerCmp(a.name) < nsPerCmp(cheap) {
			cheap = a.name
		}
		if first || nsPerCmp(a.name) > nsPerCmp(dear) {
			dear = a.name
		}
	}
	ratio := func(a, b string) (float64, float64) {
		return float64(cmps[a][0]) / float64(cmps[b][0]), ms[a][0] / ms[b][0]
	}
	fmt.Println("\n3. Random input: comparisons against the clock (comparator sorts)")
	fmt.Printf("fewest comparisons: %-22s fastest: %s\n", fewest, fastest)
	fmt.Printf("most comparisons:   %-22s slowest: %s\n", most, slowest)
	fmt.Printf("ns per comparison: %.2f (%s) to %.2f (%s), %.1fx apart\n",
		nsPerCmp(cheap), cheap, nsPerCmp(dear), dear, nsPerCmp(dear)/nsPerCmp(cheap))
	for _, a := range []string{"heapsort", "quick random", "slices.SortFunc"} {
		c, t := ratio(a, "merge top-down")
		fmt.Printf("%-16s vs merge top-down: %.2fx the comparisons, %.2fx the time\n", a, c, t)
	}
	fmt.Printf("slices.Sort (no comparator) takes %.2fx the time of slices.SortFunc\n",
		ms["slices.Sort"][0]/ms["slices.SortFunc"][0])
}
```

**Sample output:**

```
n = 1000000; comparator sorts take cmp func(a, b int) int; one call = one comparison

1. Comparisons (deterministic)
algorithm                   random      sorted    reversed    few k=10  organ-pipe
merge top-down            18673749     9884992    10066432    17950309    10475711
merge bottom-up           18716293    10095552     9884992    18008714    10384988
quick random              27939613    28648926    28547081    22753858    28489940
quick 3-way               26237748    25097244    25731046     3398870    24184744
heapsort                  36794299    37692069    36001436    33481796    37253216
introsort                 25086417    18393212    18893229    18668448    63654015
slices.SortFunc           21126865     1000011     1000011     4998649    19528383
slices.SortStableFunc     25283097     1200386     9800444     9677594     7218176
n lg n                    19931569   (the lg(n!) floor is about n lg n - 1.44n = 18488874)
every sort's output equals slices.Sort's on every shape: true

2. Milliseconds per sort (copy-only control subtracted; best of 3 rounds)
algorithm                   random      sorted    reversed    few k=10  organ-pipe ns/cmp random
merge top-down               101.0        27.9        26.7        65.7        27.9          5.41
merge bottom-up               96.1        23.9        21.8        54.2        23.6          5.14
quick random                  96.4        42.9        43.4        54.3        55.2          3.45
quick 3-way                   96.9        49.0        50.1        12.0        45.5          3.69
heapsort                     125.1        73.7        75.4        76.4        81.4          3.40
introsort                     80.4        19.1        19.9        42.7       109.0          3.20
slices.SortFunc               81.0         1.2         1.5        16.3        30.6          3.84
slices.SortStableFunc        179.6         1.8        19.7        60.0        19.7          7.10
slices.Sort                   57.6         0.5         0.7         9.2        13.9

3. Random input: comparisons against the clock (comparator sorts)
fewest comparisons: merge top-down         fastest: introsort
most comparisons:   heapsort               slowest: slices.SortStableFunc
ns per comparison: 3.20 (introsort) to 7.10 (slices.SortStableFunc), 2.2x apart
heapsort         vs merge top-down: 1.97x the comparisons, 1.24x the time
quick random     vs merge top-down: 1.50x the comparisons, 0.95x the time
slices.SortFunc  vs merge top-down: 1.13x the comparisons, 0.80x the time
slices.Sort (no comparator) takes 0.71x the time of slices.SortFunc
```

**Rust:**

```rust
use std::cmp::Ordering::{self, Greater, Less};
use std::hint::black_box;
use std::time::Instant;

// Every hand-written sort below is generic over its comparator,
// `F: FnMut(&i64, &i64) -> Ordering`, exactly like sort_unstable_by. That is
// the Rust angle of this race: the compiler makes one copy of each sort per
// comparator type (monomorphisation) and inlines the comparison, where Go
// calls a func value indirectly for every comparison. Table 1 counts calls
// with a counting closure; table 2 times a plain `a.cmp(b)` closure.
// Internals take `c: &mut F` so the recursion reuses one instantiation.

// ---- Go's math/rand/v2 PCG (PCG-DXSM) + IntN, so the pivots are Go's pivots ----
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

// ---- top-down merge sort: one scratch buffer, copy back after each merge ----

fn merge_top_down<F: FnMut(&i64, &i64) -> Ordering>(xs: &mut [i64], c: &mut F) {
    let mut buf = vec![0; xs.len()];
    merge_rec(xs, &mut buf, c);
}

fn merge_rec<F: FnMut(&i64, &i64) -> Ordering>(xs: &mut [i64], buf: &mut [i64], c: &mut F) {
    let n = xs.len();
    if n < 2 {
        return;
    }
    let mid = n / 2;
    {
        let (xl, xr) = xs.split_at_mut(mid);
        let (bl, br) = buf.split_at_mut(mid);
        merge_rec(xl, bl, c);
        merge_rec(xr, br, c);
    }
    merge(&xs[..mid], &xs[mid..], &mut buf[..n], c);
    xs.copy_from_slice(&buf[..n]);
}

/// Merges the sorted runs a and b into out. Ties take a: stable.
fn merge<F: FnMut(&i64, &i64) -> Ordering>(a: &[i64], b: &[i64], out: &mut [i64], c: &mut F) {
    let (mut i, mut j, mut k) = (0, 0, 0);
    while i < a.len() && j < b.len() {
        if c(&b[j], &a[i]) == Less {
            out[k] = b[j];
            j += 1;
        } else {
            out[k] = a[i];
            i += 1;
        }
        k += 1;
    }
    out[k..k + a.len() - i].copy_from_slice(&a[i..]);
    k += a.len() - i;
    out[k..].copy_from_slice(&b[j..]);
}

// ---- bottom-up merge sort: widths 1, 2, 4, ...; the buffers swap roles ----

fn merge_bottom_up<F: FnMut(&i64, &i64) -> Ordering>(xs: &mut [i64], c: &mut F) {
    let n = xs.len();
    let mut buf = vec![0; n];
    // One pass merges src into dst. Rust will not let `xs` and `buf` trade
    // places as two `&mut` that we later copy between, so a flag says where
    // the data is (Go swaps the two slice headers).
    let pass = |src: &[i64], dst: &mut [i64], w: usize, c: &mut F| {
        for lo in (0..n).step_by(2 * w) {
            let (mid, hi) = ((lo + w).min(n), (lo + 2 * w).min(n));
            merge(&src[lo..mid], &src[mid..hi], &mut dst[lo..hi], c);
        }
    };
    let mut in_buf = false;
    let mut w = 1;
    while w < n {
        if in_buf {
            pass(&buf, xs, w, c);
        } else {
            pass(xs, &mut buf, w, c);
        }
        in_buf = !in_buf;
        w *= 2;
    }
    if in_buf {
        xs.copy_from_slice(&buf);
    }
}

// ---- Hoare partition (CLRS), pivot = xs[0]; returns j: xs[..=j] <= p <= xs[j+1..] ----
// CLRS starts i at lo-1, an underflow for usize; here i starts at 0 and is
// stepped after each swap instead -- the same comparisons in the same order.

fn hoare<F: FnMut(&i64, &i64) -> Ordering>(xs: &mut [i64], c: &mut F) -> usize {
    let p = xs[0];
    let (mut i, mut j) = (0, xs.len());
    loop {
        j -= 1;
        while c(&xs[j], &p) == Greater {
            j -= 1;
        }
        while c(&xs[i], &p) == Less {
            i += 1;
        }
        if i >= j {
            return j;
        }
        xs.swap(i, j);
        i += 1;
    }
}

// ---- quicksort, random pivot: safe on every shape (in expectation) ----

fn quick_random<F: FnMut(&i64, &i64) -> Ordering>(mut xs: &mut [i64], c: &mut F, r: &mut Pcg) {
    while xs.len() > 1 {
        let p = r.int_n(xs.len());
        xs.swap(0, p);
        let s = hoare(xs, c);
        let (left, right) = xs.split_at_mut(s + 1);
        if left.len() < right.len() {
            // recurse on the smaller side, loop on the larger
            quick_random(left, c, r);
            xs = right;
        } else {
            quick_random(right, c, r);
            xs = left;
        }
    }
}

// ---- 3-way quicksort (Dijkstra), random pivot: one cmp call per element per pass ----

fn quick3<F: FnMut(&i64, &i64) -> Ordering>(mut xs: &mut [i64], c: &mut F, r: &mut Pcg) {
    while xs.len() > 1 {
        let p = xs[r.int_n(xs.len())];
        // xs[..lt] < p, xs[lt..i] == p, xs[gt..] > p
        let (mut lt, mut i, mut gt) = (0, 0, xs.len());
        while i < gt {
            match c(&xs[i], &p) {
                Less => {
                    xs.swap(lt, i);
                    lt += 1;
                    i += 1;
                }
                Greater => {
                    gt -= 1;
                    xs.swap(i, gt);
                }
                Ordering::Equal => i += 1,
            }
        }
        let (left, rest) = xs.split_at_mut(lt);
        let right = &mut rest[gt - lt..]; // the == p block is finished
        if left.len() < right.len() {
            quick3(left, c, r);
            xs = right;
        } else {
            quick3(right, c, r);
            xs = left;
        }
    }
}

// ---- heapsort in place (sift-down as in Go's siftDownOrdered) ----

fn heap_sort<F: FnMut(&i64, &i64) -> Ordering>(xs: &mut [i64], c: &mut F) {
    let n = xs.len();
    for i in (0..n / 2).rev() {
        sift_down(xs, i, n, c);
    }
    for end in (1..n).rev() {
        xs.swap(0, end);
        sift_down(xs, 0, end, c);
    }
}

fn sift_down<F: FnMut(&i64, &i64) -> Ordering>(
    xs: &mut [i64],
    mut root: usize,
    n: usize,
    c: &mut F,
) {
    loop {
        let mut child = 2 * root + 1; // the jump from i to 2i+1: a cache miss on big arrays
        if child >= n {
            return;
        }
        if child + 1 < n && c(&xs[child], &xs[child + 1]) == Less {
            child += 1;
        }
        if c(&xs[root], &xs[child]) != Less {
            return;
        }
        xs.swap(root, child);
        root = child;
    }
}

// ---- introsort: median-of-3 Hoare quicksort, depth 2*floor(lg n) -> heapsort, insertion <= 12 ----

fn introsort<F: FnMut(&i64, &i64) -> Ordering>(xs: &mut [i64], c: &mut F) {
    let depth = 2 * xs.len().max(1).ilog2();
    intro_rec(xs, c, depth);
}

fn intro_rec<F: FnMut(&i64, &i64) -> Ordering>(mut xs: &mut [i64], c: &mut F, mut depth: u32) {
    while xs.len() > 12 {
        if depth == 0 {
            heap_sort(xs, c); // the guarantee: Θ(n log n) whatever the pivots did
            return;
        }
        depth -= 1;
        let m = median3(xs, 0, xs.len() / 2, xs.len() - 1, c);
        xs.swap(0, m);
        let s = hoare(xs, c);
        let (left, right) = xs.split_at_mut(s + 1);
        if left.len() < right.len() {
            intro_rec(left, c, depth);
            xs = right;
        } else {
            intro_rec(right, c, depth);
            xs = left;
        }
    }
    for i in 1..xs.len() {
        // insertion sort finishes the small range
        let mut j = i;
        while j > 0 && c(&xs[j], &xs[j - 1]) == Less {
            xs.swap(j, j - 1);
            j -= 1;
        }
    }
}

fn median3<F: FnMut(&i64, &i64) -> Ordering>(
    xs: &[i64],
    mut a: usize,
    mut b: usize,
    m: usize,
    c: &mut F,
) -> usize {
    if c(&xs[b], &xs[a]) == Less {
        std::mem::swap(&mut a, &mut b); // now xs[a] <= xs[b]
    }
    if c(&xs[m], &xs[b]) == Less {
        if c(&xs[m], &xs[a]) == Less {
            return a;
        }
        return m;
    }
    b
}

// ---- the race ----

#[derive(Clone, Copy, PartialEq)]
enum Algo {
    TopDown,
    BottomUp,
    QuickRandom,
    Quick3,
    Heap,
    Intro,
    SortUnstableBy,
    SortBy,
    SortUnstable, // takes no comparator
}

const ALGOS: [(Algo, &str); 9] = [
    (Algo::TopDown, "merge top-down"),
    (Algo::BottomUp, "merge bottom-up"),
    (Algo::QuickRandom, "quick random"),
    (Algo::Quick3, "quick 3-way"),
    (Algo::Heap, "heapsort"),
    (Algo::Intro, "introsort"),
    (Algo::SortUnstableBy, "sort_unstable_by"),
    (Algo::SortBy, "sort_by"),
    (Algo::SortUnstable, "sort_unstable"),
];

/// One `match`, one monomorphised copy of every sort per comparator type.
fn run<F: FnMut(&i64, &i64) -> Ordering>(a: Algo, xs: &mut [i64], mut c: F) {
    match a {
        Algo::TopDown => merge_top_down(xs, &mut c),
        Algo::BottomUp => merge_bottom_up(xs, &mut c),
        Algo::QuickRandom => quick_random(xs, &mut c, &mut Pcg::new(3, 4)),
        Algo::Quick3 => quick3(xs, &mut c, &mut Pcg::new(3, 4)),
        Algo::Heap => heap_sort(xs, &mut c),
        Algo::Intro => introsort(xs, &mut c),
        Algo::SortUnstableBy => xs.sort_unstable_by(c),
        Algo::SortBy => xs.sort_by(c),
        Algo::SortUnstable => xs.sort_unstable(),
    }
}

/// Copies the pristine input into work INSIDE the timed run (sorting sorted
/// data would measure nothing) and returns milliseconds.
fn ms_once(work: &mut [i64], pristine: &[i64], f: impl FnOnce(&mut [i64])) -> f64 {
    let t = Instant::now();
    work.copy_from_slice(black_box(pristine));
    f(work);
    black_box(&mut *work);
    t.elapsed().as_nanos() as f64 / 1e6
}

fn main() {
    const N: usize = 1_000_000;
    let mut r = Pcg::new(1, 2);
    let mut random: Vec<i64> = (0..N as i64).collect();
    for i in (1..N).rev() {
        // Fisher-Yates over IntN, in the same order as the Go program
        random.swap(i, r.int_n(i + 1));
    }
    let sorted: Vec<i64> = (0..N as i64).collect();
    let reversed: Vec<i64> = sorted.iter().rev().copied().collect();
    let few: Vec<i64> = (0..N).map(|_| r.int_n(10) as i64).collect();
    let organ: Vec<i64> = (0..N).map(|i| i.min(N - 1 - i) as i64).collect();
    let shapes: [(&str, &[i64]); 5] = [
        ("random", &random),
        ("sorted", &sorted),
        ("reversed", &reversed),
        ("few k=10", &few),
        ("organ-pipe", &organ),
    ];

    println!(
        "n = {N}; comparator sorts take F: FnMut(&i64, &i64) -> Ordering; one call = one comparison\n"
    );
    println!("1. Comparisons (deterministic)");
    let header = || {
        print!("{:<22}", "algorithm");
        for (name, _) in &shapes {
            print!(" {name:>11}");
        }
    };
    header();
    println!();
    let (mut work, mut want) = (vec![0; N], vec![0; N]);
    let mut cmps: Vec<Vec<u64>> = Vec::new();
    let mut all_ok = true;
    for &(a, name) in &ALGOS[..ALGOS.len() - 1] {
        print!("{name:<22}");
        let mut row = Vec::new();
        for &(_, xs) in &shapes {
            want.copy_from_slice(xs);
            want.sort_unstable();
            work.copy_from_slice(xs);
            let mut count = 0u64;
            run(a, &mut work, |x, y| {
                count += 1;
                x.cmp(y)
            });
            all_ok &= work == want;
            row.push(count);
            print!(" {count:>11}");
        }
        cmps.push(row);
        println!();
    }
    let lgn = (N as f64).log2();
    let nf = N as f64;
    println!(
        "{:<22} {:>11.0}   (the lg(n!) floor is about n lg n - 1.44n = {:.0})",
        "n lg n",
        nf * lgn,
        nf * lgn - std::f64::consts::LOG2_E * nf
    );
    println!("every sort's output equals sort_unstable's on every shape: {all_ok}");

    // Best of 3 ROUNDS, each round timing every sort on every shape once: a
    // burst of load on a shared machine then spoils one round of several
    // sorts, not all three tries of one sort.
    let mut ms = vec![[f64::INFINITY; 5]; ALGOS.len()];
    let mut ctrl = [f64::INFINITY; 5];
    for _ in 0..3 {
        for (si, &(_, xs)) in shapes.iter().enumerate() {
            ctrl[si] = ctrl[si].min(ms_once(&mut work, xs, |_| {})); // copy-only control
            for (ai, &(a, _)) in ALGOS.iter().enumerate() {
                let t = ms_once(&mut work, xs, |w| run(a, w, |x, y| x.cmp(y)));
                ms[ai][si] = ms[ai][si].min(t);
            }
        }
    }
    println!("\n2. Milliseconds per sort (copy-only control subtracted; best of 3 rounds)");
    header();
    println!(" {:>13}", "ns/cmp random");
    for (ai, &(_, name)) in ALGOS.iter().enumerate() {
        print!("{name:<22}");
        for (t, c) in ms[ai].iter_mut().zip(ctrl) {
            *t = (*t - c).max(0.0);
            print!(" {t:>11.1}");
        }
        if let Some(c) = cmps.get(ai) {
            print!(" {:>13.2}", ms[ai][0] * 1e6 / c[0] as f64);
        }
        println!();
    }

    // What the two tables say, computed from this run's numbers.
    let idx = |name: &str| ALGOS.iter().position(|&(_, n)| n == name).unwrap();
    let comparator = 0..ALGOS.len() - 1; // every row except sort_unstable
    let ns_per_cmp = |i: usize| ms[i][0] * 1e6 / cmps[i][0] as f64;
    let by_cmps = |i: &usize, j: &usize| cmps[*i][0].cmp(&cmps[*j][0]);
    let by_ms = |i: &usize, j: &usize| ms[*i][0].total_cmp(&ms[*j][0]);
    let by_ns = |i: &usize, j: &usize| ns_per_cmp(*i).total_cmp(&ns_per_cmp(*j));
    let name = |i: Option<usize>| ALGOS[i.unwrap()].1;
    let (cheap, dear) = (
        comparator.clone().min_by(by_ns).unwrap(),
        comparator.clone().max_by(by_ns).unwrap(),
    );
    println!("\n3. Random input: comparisons against the clock (comparator sorts)");
    println!(
        "fewest comparisons: {:<22} fastest: {}",
        name(comparator.clone().min_by(by_cmps)),
        name(comparator.clone().min_by(by_ms))
    );
    println!(
        "most comparisons:   {:<22} slowest: {}",
        name(comparator.clone().max_by(by_cmps)),
        name(comparator.clone().max_by(by_ms))
    );
    println!(
        "ns per comparison: {:.2} ({}) to {:.2} ({}), {:.1}x apart",
        ns_per_cmp(cheap),
        ALGOS[cheap].1,
        ns_per_cmp(dear),
        ALGOS[dear].1,
        ns_per_cmp(dear) / ns_per_cmp(cheap)
    );
    let base = idx("merge top-down");
    for name in ["heapsort", "quick random", "sort_unstable_by"] {
        let i = idx(name);
        println!(
            "{name:<16} vs merge top-down: {:.2}x the comparisons, {:.2}x the time",
            cmps[i][0] as f64 / cmps[base][0] as f64,
            ms[i][0] / ms[base][0]
        );
    }
    println!(
        "sort_unstable (no comparator) takes {:.2}x the time of sort_unstable_by",
        ms[idx("sort_unstable")][0] / ms[idx("sort_unstable_by")][0]
    );
}
```

**Rust sample output:**

```
n = 1000000; comparator sorts take F: FnMut(&i64, &i64) -> Ordering; one call = one comparison

1. Comparisons (deterministic)
algorithm                   random      sorted    reversed    few k=10  organ-pipe
merge top-down            18673749     9884992    10066432    17950309    10475711
merge bottom-up           18716293    10095552     9884992    18008714    10384988
quick random              27939613    28648926    28547081    22753858    28489940
quick 3-way               26237748    25097244    25731046     3398870    24184744
heapsort                  36794299    37692069    36001436    33481796    37253216
introsort                 25086417    18393212    18893229    18668448    63654015
sort_unstable_by          20561124      999999      999999     4510535    26874648
sort_by                   20822755      999999      999999     4511942     1999998
n lg n                    19931569   (the lg(n!) floor is about n lg n - 1.44n = 18488874)
every sort's output equals sort_unstable's on every shape: true

2. Milliseconds per sort (copy-only control subtracted; best of 3 rounds)
algorithm                   random      sorted    reversed    few k=10  organ-pipe ns/cmp random
merge top-down                67.2        16.8        16.3        33.6        16.9          3.60
merge bottom-up               60.9        11.3        10.8        27.4        11.2          3.25
quick random                  60.0        23.2        22.2        21.9        28.8          2.15
quick 3-way                   74.3        33.4        33.3         9.4        30.8          2.83
heapsort                      73.6        32.7        35.3        32.4        36.5          2.00
introsort                     48.1         5.6         6.1        16.3        32.8          1.92
sort_unstable_by              10.2         0.3         0.4         1.8        10.3          0.50
sort_by                       12.9         0.3         0.4         2.8         3.2          0.62
sort_unstable                 10.1         0.2         0.3         1.9        10.3

3. Random input: comparisons against the clock (comparator sorts)
fewest comparisons: merge top-down         fastest: sort_unstable_by
most comparisons:   heapsort               slowest: quick 3-way
ns per comparison: 0.50 (sort_unstable_by) to 3.60 (merge top-down), 7.3x apart
heapsort         vs merge top-down: 1.97x the comparisons, 1.10x the time
quick random     vs merge top-down: 1.50x the comparisons, 0.89x the time
sort_unstable_by vs merge top-down: 1.10x the comparisons, 0.15x the time
sort_unstable (no comparator) takes 0.99x the time of sort_unstable_by
```

> **Go vs Rust:** The six hand-written sorts give identical comparison counts in both languages (same code, same PCG pivots); the standard-library rows differ by design. Rust's ipnsort and driftsort detect a whole-slice run, so sorted and reversed input cost exactly n − 1 = 999,999 comparisons (Go's pdqsort: 1,000,011), and `sort_by` sorts organ-pipe as two runs in 1,999,998. On the clock, `sort_unstable_by` took about 10 ms on random input against about 81 ms for `slices.SortFunc` (~8×), and it tied `sort_unstable` because the closure is monomorphised and inlined. The same hand-written code ran 1.3–1.7× faster in Rust, and Rust's heapsort took only about 1.1× merge's time (Go: 1.15–1.4×), over four runs.

**Complexity:** every sort here is Θ(n log n) on these shapes; introsort's organ-pipe row is its heapsort fallback firing (63.7 M comparisons, 3.2 n lg n). The comparison counts are deterministic: on random input top-down merge makes 18.67 M (1.010 × lg(n!)), heapsort 1.97× that and the random-pivot quicksort 1.50×. The clock is not deterministic. Over five Go runs on a loaded machine (best of 3 interleaved rounds each), heapsort took 1.15–1.4× merge's time and the random-pivot quicksort 0.85–1.05×, so twice the comparisons cost only 15–40% more time. The fastest comparator sort was `slices.SortFunc` or introsort (80–90 ms), and the slowest was `slices.SortStableFunc` (about 175–190 ms). One comparison cost between about 3.1 and 7.5 ns depending on the sort, a 2.2–2.3× spread. `slices.Sort` took about 0.7× `SortFunc`'s time. It is the same pdqsort, generated twice in the source (read, not run): `zsortordered.go` compares with `cmp.Less`, which the compiler can inline, and `zsortanyfunc.go` calls `cmp` through a func value. The usual cache-miss argument against heapsort is weak at this size: 8 MB of ints fits in this M4's 16 MB L2 (plausible mechanism, not measured).

---

## 16. Capstone: `Sort`, `SortStable`, `Select`, built, proved, measured

`🔴 hard` · *Applied build / oracle-proved*

This example packages the lesson as three generic functions with `slices.SortFunc`-style signatures.
`Sort[E](x, cmp)` is an introsort. It checks the whole slice for one run first, which makes sorted and strictly
descending input Θ(n). Then it runs quicksort with Tukey's ninther and Dijkstra's 3-way partition, insertion sort
at ≤ 12 elements, and a heapsort fallback once a range has had `bits.Len(n)` bad splits, which keeps it
Θ(n log n) in the worst case. `SortStable[E]` is a bottom-up natural merge sort with one buffer. `Select[E](x, k,
cmp)` is iterative quickselect with the same pivot, partition and heapsort fallback: Θ(n) expected and
Θ(n log n) worst case. A median-of-medians fallback would make the worst case Θ(n); it is not built here. The
standard library is the oracle. 20,000 seeded inputs, including duplicate-heavy and adversarial shapes, are
sorted as tagged records, so stability is checked too. McIlroy's adversary is run against all three functions,
and then they are timed against the standard library.

**Steps:**

1. Build `Sort`, `SortStable` and `Select` on `cmp func(a, b E) int`. They share the pivot, partition, insertion
   sort and heapsort helpers.
2. Prove them on 20,000 PCG trials against `slices.SortFunc` and `slices.SortStableFunc`, checking keys,
   permutation, tags and the selection property. Run McIlroy's adversary, then count comparisons on sorted,
   reversed and all-equal input.
3. Time all three against the standard library at n = 10⁶ on random and sorted input: best of 3 interleaved
   rounds, with a copy-only control subtracted.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/bits"
	"math/rand/v2"
	"slices"
	"time"
)

// ===================== the package: Sort, SortStable, Select =====================

// Sort sorts x in ascending order as determined by cmp. It is not stable.
// Introsort: a whole-slice run check, then quicksort with a ninther pivot and
// a 3-way partition, insertion sort at <= 12, heapsort after too many bad splits.
func Sort[E any](x []E, cmp func(a, b E) int) { sortCounted(x, cmp) }

// sortCounted is Sort, reporting how many ranges fell back to heapsort.
func sortCounted[E any](x []E, cmp func(a, b E) int) (fallbacks int) {
	n := len(x)
	if n < 2 {
		return 0
	}
	// One run check over the whole slice, as Rust's ipnsort does: sorted input
	// costs n-1 comparisons, strictly descending input n-1 and a reverse.
	desc := cmp(x[1], x[0]) < 0
	i := 2
	for i < n && (cmp(x[i], x[i-1]) < 0) == desc {
		i++
	}
	if i == n {
		if desc {
			slices.Reverse(x)
		}
		return 0
	}
	introsort(x, cmp, bits.Len(uint(n)), &fallbacks)
	return fallbacks
}

func introsort[E any](x []E, cmp func(a, b E) int, limit int, fallbacks *int) {
	for len(x) > 12 {
		if limit == 0 {
			heapSort(x, cmp) // the guarantee: Θ(n log n) whatever the pivots did
			*fallbacks++
			return
		}
		pivotToFront(x, cmp)
		lt, gt := partition3(x, cmp)
		left, right := x[:lt], x[gt:] // x[lt:gt] == pivot: finished
		if max(len(left), len(right)) > len(x)-len(x)/8 {
			limit-- // a bad split: less than 1/8 of the range was settled
		}
		if len(left) < len(right) { // recurse on the smaller side: depth <= lg n
			introsort(left, cmp, limit, fallbacks)
			x = right
		} else {
			introsort(right, cmp, limit, fallbacks)
			x = left
		}
	}
	insertionSort(x, cmp)
}

// pivotToFront moves the pivot to x[0]: Tukey's ninther from 50 elements,
// median of 3 below that (len(x) > 12 here).
func pivotToFront[E any](x []E, cmp func(a, b E) int) {
	n, m := len(x), len(x)/2
	if n >= 50 {
		s := n / 8
		m = median3(x, median3(x, 0, s, 2*s, cmp), median3(x, m-s, m, m+s, cmp),
			median3(x, n-1-2*s, n-1-s, n-1, cmp), cmp)
	} else {
		m = median3(x, 0, m, n-1, cmp)
	}
	x[0], x[m] = x[m], x[0]
}

func median3[E any](x []E, a, b, c int, cmp func(a, b E) int) int {
	if cmp(x[b], x[a]) < 0 {
		a, b = b, a // now x[a] <= x[b]
	}
	if cmp(x[c], x[b]) < 0 {
		if cmp(x[c], x[a]) < 0 {
			return a
		}
		return c
	}
	return b
}

// partition3 (Dijkstra) with the pivot parked at x[0]: one cmp call per
// element. Returns lt, gt with x[:lt] < p, x[lt:gt] == p, x[gt:] > p.
func partition3[E any](x []E, cmp func(a, b E) int) (int, int) {
	lt, i, gt := 1, 1, len(x)
	for i < gt {
		switch c := cmp(x[i], x[0]); {
		case c < 0:
			x[lt], x[i] = x[i], x[lt]
			lt++
			i++
		case c > 0:
			gt--
			x[i], x[gt] = x[gt], x[i]
		default:
			i++
		}
	}
	x[0], x[lt-1] = x[lt-1], x[0] // the pivot joins the == block
	return lt - 1, gt
}

func insertionSort[E any](x []E, cmp func(a, b E) int) {
	for i := 1; i < len(x); i++ {
		for j := i; j > 0 && cmp(x[j], x[j-1]) < 0; j-- {
			x[j], x[j-1] = x[j-1], x[j]
		}
	}
}

func heapSort[E any](x []E, cmp func(a, b E) int) {
	n := len(x)
	for i := n/2 - 1; i >= 0; i-- {
		siftDown(x, i, n, cmp)
	}
	for end := n - 1; end > 0; end-- {
		x[0], x[end] = x[end], x[0]
		siftDown(x, 0, end, cmp)
	}
}

func siftDown[E any](x []E, root, n int, cmp func(a, b E) int) {
	for {
		child := 2*root + 1
		if child >= n {
			return
		}
		if child+1 < n && cmp(x[child], x[child+1]) < 0 {
			child++
		}
		if cmp(x[root], x[child]) >= 0 {
			return
		}
		x[root], x[child] = x[child], x[root]
		root = child
	}
}

// SortStable sorts x, keeping equal elements in their input order. Bottom-up
// natural merge sort: find the runs, then merge neighbours pass by pass,
// ping-ponging between x and ONE buffer of len(x) (plus the run boundaries).
func SortStable[E any](x []E, cmp func(a, b E) int) {
	n := len(x)
	if n < 2 {
		return
	}
	bounds := []int{0} // run i is x[bounds[i]:bounds[i+1]]
	for i := 0; i < n; {
		j := i + 1
		if j < n && cmp(x[j], x[i]) < 0 {
			// Strictly descending: reverse it. Strict, so no two equal
			// elements trade places, and the sort stays stable.
			j++
			for j < n && cmp(x[j], x[j-1]) < 0 {
				j++
			}
			slices.Reverse(x[i:j])
		} else if j < n { // non-descending: that first comparison is already done
			j++
			for j < n && cmp(x[j], x[j-1]) >= 0 {
				j++
			}
		}
		bounds = append(bounds, j)
		i = j
	}
	src, dst := x, make([]E, n)
	inBuf := false
	for len(bounds) > 2 { // more than one run left
		w := 1 // rewrite bounds in place: writes trail reads
		for k := 0; k+1 < len(bounds); k += 2 {
			lo, mid := bounds[k], bounds[k+1]
			hi := mid
			if k+2 < len(bounds) {
				hi = bounds[k+2]
				merge(src[lo:mid], src[mid:hi], dst[lo:hi], cmp)
			} else {
				copy(dst[lo:hi], src[lo:hi]) // the odd run out
			}
			bounds[w] = hi
			w++
		}
		bounds = bounds[:w]
		src, dst = dst, src
		inBuf = !inBuf
	}
	if inBuf {
		copy(x, src)
	}
}

// merge merges sorted a and b into out. Ties take a: that is the stability.
func merge[E any](a, b, out []E, cmp func(a, b E) int) {
	i, j, k := 0, 0, 0
	for i < len(a) && j < len(b) {
		if cmp(b[j], a[i]) < 0 {
			out[k] = b[j]
			j++
		} else {
			out[k] = a[i]
			i++
		}
		k++
	}
	k += copy(out[k:], a[i:])
	copy(out[k:], b[j:])
}

// Select reorders x so that x[k] is the element a sort would put there, with
// x[:k] <= x[k] <= x[k+1:]. Iterative quickselect (ninther, 3-way partition,
// insertion sort at <= 12); after bits.Len(n) bad splits the remaining range
// is heapsorted, so the worst case is Θ(n log n), not Θ(n²).
func Select[E any](x []E, k int, cmp func(a, b E) int) { selectCounted(x, k, cmp) }

func selectCounted[E any](x []E, k int, cmp func(a, b E) int) (fellBack bool) {
	if k < 0 || k >= len(x) {
		panic("Select: k out of range")
	}
	lo, hi, limit := 0, len(x), bits.Len(uint(len(x)))
	for hi-lo > 12 {
		r := x[lo:hi]
		if limit == 0 {
			heapSort(r, cmp)
			return true
		}
		pivotToFront(r, cmp)
		lt, gt := partition3(r, cmp)
		if max(lt, len(r)-gt) > len(r)-len(r)/8 {
			limit--
		}
		switch {
		case k-lo < lt:
			hi = lo + lt
		case k-lo >= gt:
			lo += gt
		default:
			return false // k landed in the == block: done
		}
	}
	insertionSort(x[lo:hi], cmp)
	return false
}

// ===================== the proof, the adversary, the measurement =====================

type rec struct{ key, tag int } // tag = input position: shows stability

func byKey(a, b rec) int { return cmp.Compare(a.key, b.key) }

func byKeyTag(a, b rec) int {
	if c := cmp.Compare(a.key, b.key); c != 0 {
		return c
	}
	return cmp.Compare(a.tag, b.tag)
}

// adversary is McIlroy's: values are decided lazily, and whatever looks like
// the pivot is frozen as small as possible.
type adversary struct {
	val                         []int
	gas, solid, candidate, cmps int
}

func newAdversary(n int) *adversary {
	return &adversary{val: slices.Repeat([]int{n}, n), gas: n}
}

func (a *adversary) cmp(x, y int) int {
	a.cmps++
	if a.val[x] == a.gas && a.val[y] == a.gas {
		if x == a.candidate {
			a.val[x], a.solid = a.solid, a.solid+1
		} else {
			a.val[y], a.solid = a.solid, a.solid+1
		}
	}
	if a.val[x] == a.gas {
		a.candidate = x
	} else if a.val[y] == a.gas {
		a.candidate = y
	}
	return cmp.Compare(a.val[x], a.val[y])
}

func msOnce(work, pristine []int, f func([]int)) float64 {
	start := time.Now()
	copy(work, pristine) // a fresh copy INSIDE the timed run
	f(work)
	return float64(time.Since(start).Nanoseconds()) / 1e6
}

func main() {
	r := rand.New(rand.NewPCG(1, 2))
	const trials = 20000
	fmt.Printf("1. Proof against the standard library: %d trials, n = IntN(300), 7 shapes\n", trials)
	fmt.Println("   (keys IntN(8), keys IntN(2^30), sorted, reversed, all-equal, organ-pipe, sawtooth)")
	shapes := []func(i, n int) int{
		func(i, n int) int { return r.IntN(8) },       // narrow: many duplicates
		func(i, n int) int { return r.IntN(1 << 30) }, // wide: almost all distinct
		func(i, n int) int { return i },               // sorted
		func(i, n int) int { return n - 1 - i },       // reversed
		func(i, n int) int { return 7 },               // all-equal
		func(i, n int) int { return min(i, n-1-i) },   // organ-pipe
		func(i, n int) int { return i % 64 },          // sawtooth
	}
	var bad [4]int
	reordered := 0
	for t := range trials {
		n := r.IntN(300)
		in := make([]rec, n)
		for i := range in {
			in[i] = rec{shapes[t%7](i, n), i}
		}
		// Since tags rise in input order, the stable oracle is also the input
		// sorted by (key, tag): the yardstick for "is a permutation".
		want := slices.Clone(in)
		slices.SortStableFunc(want, byKey)
		oracle := slices.Clone(in)
		slices.SortFunc(oracle, byKey)

		got := slices.Clone(in)
		Sort(got, byKey)
		if !slices.EqualFunc(got, oracle, func(a, b rec) bool { return a.key == b.key }) {
			bad[0]++
		}
		if !slices.Equal(got, want) {
			reordered++ // equal keys came out in a different order: legal for Sort
		}
		slices.SortFunc(got, byKeyTag)
		if !slices.Equal(got, want) {
			bad[1]++
		}
		st := slices.Clone(in)
		SortStable(st, byKey)
		if !slices.Equal(st, want) {
			bad[2]++
		}
		if n > 0 {
			k := r.IntN(n)
			s := slices.Clone(in)
			Select(s, k, byKey)
			ok := s[k].key == oracle[k].key
			for i, e := range s {
				ok = ok && (i >= k || e.key <= s[k].key) && (i <= k || e.key >= s[k].key)
			}
			slices.SortFunc(s, byKeyTag)
			if !ok || !slices.Equal(s, want) {
				bad[3]++
			}
		}
	}
	for i, check := range []string{
		"Sort       keys == slices.SortFunc's",
		"Sort       output is a permutation of the input",
		"SortStable == slices.SortStableFunc's, tags too",
		"Select(k)  == sorted[k], left <= kth <= right",
	} {
		fmt.Printf("   %-48s %d mismatches in %d trials\n", check, bad[i], trials)
	}
	fmt.Printf("   (Sort reordered equal keys in %d trials: it is not stable, so the tag check has teeth)\n", reordered)

	const n = 100_000
	nlgn := n * math.Log2(n)
	fmt.Printf("\n2. McIlroy's adversary, n = %d (n lg n = %.0f)\n", n, nlgn)
	// The adversary sorts the indices 0..n-1; values exist only as its answers.
	// run returns a note and whether the result is right (val is final then).
	attack := func(name string, run func(idx []int, c func(a, b int) int, val []int) (string, bool)) int {
		adv := newAdversary(n)
		idx := make([]int, n)
		for i := range idx {
			idx[i] = i
		}
		note, ok := run(idx, adv.cmp, adv.val)
		fmt.Printf("   %-22s %8d comparisons = %4.2f n lg n = %5.2f n, %s, correct: %v\n",
			name, adv.cmps, float64(adv.cmps)/nlgn, float64(adv.cmps)/n, note, ok)
		return adv.cmps
	}
	sortedBy := func(idx, val []int) bool {
		return slices.IsSortedFunc(idx, func(x, y int) int { return cmp.Compare(val[x], val[y]) })
	}
	shipped := attack("Sort", func(idx []int, c func(a, b int) int, val []int) (string, bool) {
		fb := sortCounted(idx, c)
		return fmt.Sprintf("fallbacks %d", fb), sortedBy(idx, val)
	})
	core := attack("Sort, run check off", func(idx []int, c func(a, b int) int, val []int) (string, bool) {
		fb := 0
		introsort(idx, c, bits.Len(uint(n)), &fb)
		return fmt.Sprintf("fallbacks %d", fb), sortedBy(idx, val)
	})
	attack("Select(n/2)", func(idx []int, c func(a, b int) int, val []int) (string, bool) {
		fell := selectCounted(idx, n/2, c)
		ok := true
		for i, e := range idx {
			ok = ok && (i >= n/2 || val[e] <= val[idx[n/2]]) && (i <= n/2 || val[e] >= val[idx[n/2]])
		}
		return fmt.Sprintf("fallback %v", fell), ok
	})
	if shipped == n-1 {
		fmt.Println("   Sort as shipped made n-1 comparisons: its run check goes first, the adversary's")
		fmt.Println("   lazy answers to it formed one ascending run, and the quicksort never started.")
	}
	fmt.Printf("   the quicksort core stays under 3 n lg n: %v (the heapsort fallback caps it)\n", float64(core) <= 3*nlgn)

	fmt.Printf("\n3. Comparison counts, n = %d\n", n)
	countOf := func(xs []int, f func([]int, func(a, b int) int)) int {
		c := 0
		f(slices.Clone(xs), func(a, b int) int { c++; return cmp.Compare(a, b) })
		return c
	}
	random, sorted, reversed, equal := make([]int, n), make([]int, n), make([]int, n), make([]int, n)
	for i := range n {
		random[i], sorted[i], reversed[i] = r.IntN(n), i, n-1-i
	}
	fmt.Printf("   %-10s %10s %10s %10s %10s\n", "", "random", "sorted", "reversed", "all-equal")
	within := true
	for _, row := range []struct {
		name string
		f    func([]int, func(a, b int) int)
	}{{"Sort", Sort[int]}, {"SortStable", SortStable[int]}} {
		fmt.Printf("   %-10s", row.name)
		for i, xs := range [][]int{random, sorted, reversed, equal} {
			c := countOf(xs, row.f)
			fmt.Printf(" %10d", c)
			within = within && (i == 0 || c <= 2*n)
		}
		fmt.Println()
	}
	fmt.Println("   sorted, reversed and all-equal input cost <= 2n for both:", within)

	const big = 1_000_000
	fmt.Printf("\n4. Milliseconds per call, n = %d (copy-only control subtracted; best of 3 rounds)\n", big)
	bigRandom, bigSorted := make([]int, big), make([]int, big)
	for i := range big {
		bigRandom[i], bigSorted[i] = r.IntN(big), i
	}
	c := cmp.Compare[int]
	rows := []struct {
		name string
		f    func([]int)
	}{
		{"Sort", func(x []int) { Sort(x, c) }},
		{"slices.SortFunc", func(x []int) { slices.SortFunc(x, c) }},
		{"slices.Sort", slices.Sort[[]int]},
		{"SortStable", func(x []int) { SortStable(x, c) }},
		{"slices.SortStableFunc", func(x []int) { slices.SortStableFunc(x, c) }},
		{"Select(n/2)", func(x []int) { Select(x, big/2, c) }},
		{"SortFunc, then x[n/2]", func(x []int) { slices.SortFunc(x, c); _ = x[big/2] }},
	}
	inputs := [][]int{bigRandom, bigSorted}
	inf := math.Inf(1)
	ms, ctrl := slices.Repeat([][2]float64{{inf, inf}}, len(rows)), [2]float64{inf, inf}
	work := make([]int, big)
	for range 3 { // rounds interleaved: a burst of load spoils one round, not one row
		for si, in := range inputs {
			ctrl[si] = min(ctrl[si], msOnce(work, in, func([]int) {}))
			for i, row := range rows {
				ms[i][si] = min(ms[i][si], msOnce(work, in, row.f))
			}
		}
	}
	fmt.Printf("   %-22s %8s %8s\n", "", "random", "sorted")
	for i, row := range rows {
		ms[i][0], ms[i][1] = max(ms[i][0]-ctrl[0], 0), max(ms[i][1]-ctrl[1], 0)
		fmt.Printf("   %-22s %8.1f %8.1f\n", row.name, ms[i][0], ms[i][1])
	}
	fmt.Printf("   random input: Sort takes %.2fx slices.SortFunc's time, SortStable %.2fx slices.SortStableFunc's,\n",
		ms[0][0]/ms[1][0], ms[3][0]/ms[4][0])
	fmt.Printf("   and Select(n/2) %.2fx the time of sorting and indexing\n", ms[5][0]/ms[6][0])
}
```

**Sample output:**

```
1. Proof against the standard library: 20000 trials, n = IntN(300), 7 shapes
   (keys IntN(8), keys IntN(2^30), sorted, reversed, all-equal, organ-pipe, sawtooth)
   Sort       keys == slices.SortFunc's             0 mismatches in 20000 trials
   Sort       output is a permutation of the input  0 mismatches in 20000 trials
   SortStable == slices.SortStableFunc's, tags too  0 mismatches in 20000 trials
   Select(k)  == sorted[k], left <= kth <= right    0 mismatches in 20000 trials
   (Sort reordered equal keys in 7731 trials: it is not stable, so the tag check has teeth)

2. McIlroy's adversary, n = 100000 (n lg n = 1660964)
   Sort                      99999 comparisons = 0.06 n lg n =  1.00 n, fallbacks 0, correct: true
   Sort, run check off     4626056 comparisons = 2.79 n lg n = 46.26 n, fallbacks 1, correct: true
   Select(n/2)             4625958 comparisons = 2.79 n lg n = 46.26 n, fallback true, correct: true
   Sort as shipped made n-1 comparisons: its run check goes first, the adversary's
   lazy answers to it formed one ascending run, and the quicksort never started.
   the quicksort core stays under 3 n lg n: true (the heapsort fallback caps it)

3. Comparison counts, n = 100000
                  random     sorted   reversed  all-equal
   Sort          1672058      99999      99999      99999
   SortStable    1610136      99999      99999      99999
   sorted, reversed and all-equal input cost <= 2n for both: true

4. Milliseconds per call, n = 1000000 (copy-only control subtracted; best of 3 rounds)
                            random   sorted
   Sort                       90.0      1.2
   slices.SortFunc            85.6      1.2
   slices.Sort                58.2      0.4
   SortStable                 97.2      1.3
   slices.SortStableFunc     184.4      2.2
   Select(n/2)                 8.8      1.6
   SortFunc, then x[n/2]      83.4      1.5
   random input: Sort takes 1.05x slices.SortFunc's time, SortStable 0.53x slices.SortStableFunc's,
   and Select(n/2) 0.11x the time of sorting and indexing
```

**Rust:**

```rust
use std::cmp::Ordering::{self, Equal, Greater, Less};
use std::hint::black_box;
use std::time::Instant;

// ===================== the package: sort, sort_stable, select =====================
// Public functions take `F: FnMut(&T, &T) -> Ordering` by value, like
// sort_unstable_by; the internals take `&mut F` so recursion reuses one
// instantiation. `sort` and `select` only swap, so they need no bound on T;
// `sort_stable` copies into a buffer, so it needs `T: Clone`.

/// Sorts x by cmp. Not stable. Introsort: a whole-slice run check, then
/// quicksort with a ninther pivot and a 3-way partition, insertion sort at
/// <= 12, heapsort after too many bad splits.
fn sort<T, F: FnMut(&T, &T) -> Ordering>(x: &mut [T], mut cmp: F) {
    sort_counted(x, &mut cmp);
}

/// `sort`, reporting how many ranges fell back to heapsort.
fn sort_counted<T, F: FnMut(&T, &T) -> Ordering>(x: &mut [T], cmp: &mut F) -> u32 {
    let n = x.len();
    if n < 2 {
        return 0;
    }
    // One run check over the whole slice, as ipnsort does: sorted input costs
    // n-1 comparisons, strictly descending input n-1 and a reverse.
    let desc = cmp(&x[1], &x[0]) == Less;
    let mut i = 2;
    while i < n && (cmp(&x[i], &x[i - 1]) == Less) == desc {
        i += 1;
    }
    if i == n {
        if desc {
            x.reverse();
        }
        return 0;
    }
    let mut fallbacks = 0;
    introsort(x, cmp, usize::BITS - n.leading_zeros(), &mut fallbacks); // bits.Len(n)
    fallbacks
}

fn introsort<T, F: FnMut(&T, &T) -> Ordering>(
    mut x: &mut [T],
    cmp: &mut F,
    mut limit: u32,
    fallbacks: &mut u32,
) {
    while x.len() > 12 {
        if limit == 0 {
            heap_sort(x, cmp); // the guarantee: Θ(n log n) whatever the pivots did
            *fallbacks += 1;
            return;
        }
        pivot_to_front(x, cmp);
        let (lt, gt) = partition3(x, cmp);
        let len = x.len();
        if lt.max(len - gt) > len - len / 8 {
            limit -= 1; // a bad split: less than 1/8 of the range was settled
        }
        let (left, rest) = x.split_at_mut(lt);
        let right = &mut rest[gt - lt..]; // x[lt..gt] == pivot: finished
        if left.len() < right.len() {
            // recurse on the smaller side: depth <= lg n, fine on a fixed stack
            introsort(left, cmp, limit, fallbacks);
            x = right;
        } else {
            introsort(right, cmp, limit, fallbacks);
            x = left;
        }
    }
    insertion_sort(x, cmp);
}

/// Moves the pivot to x[0]: Tukey's ninther from 50 elements, median of 3
/// below that (x.len() > 12 here).
fn pivot_to_front<T, F: FnMut(&T, &T) -> Ordering>(x: &mut [T], cmp: &mut F) {
    let (n, mut m) = (x.len(), x.len() / 2);
    if n >= 50 {
        let s = n / 8;
        let a = median3(x, 0, s, 2 * s, cmp);
        let b = median3(x, m - s, m, m + s, cmp);
        let c = median3(x, n - 1 - 2 * s, n - 1 - s, n - 1, cmp);
        m = median3(x, a, b, c, cmp);
    } else {
        m = median3(x, 0, m, n - 1, cmp);
    }
    x.swap(0, m);
}

fn median3<T, F: FnMut(&T, &T) -> Ordering>(
    x: &[T],
    mut a: usize,
    mut b: usize,
    c: usize,
    cmp: &mut F,
) -> usize {
    if cmp(&x[b], &x[a]) == Less {
        std::mem::swap(&mut a, &mut b); // now x[a] <= x[b]
    }
    if cmp(&x[c], &x[b]) == Less {
        if cmp(&x[c], &x[a]) == Less {
            return a;
        }
        return c;
    }
    b
}

/// Dijkstra's 3-way partition with the pivot parked at x[0]: one cmp call
/// per element. Returns (lt, gt): x[..lt] < p, x[lt..gt] == p, x[gt..] > p.
/// split_first_mut lends the pivot (shared) and the rest (mutable) at once,
/// so no copy of the pivot is needed -- and T needs no Clone.
fn partition3<T, F: FnMut(&T, &T) -> Ordering>(x: &mut [T], cmp: &mut F) -> (usize, usize) {
    let (pivot, rest) = x.split_first_mut().unwrap();
    let (mut lt, mut i, mut gt) = (0, 0, rest.len()); // indices into rest = x[1..]
    while i < gt {
        match cmp(&rest[i], pivot) {
            Less => {
                rest.swap(lt, i);
                lt += 1;
                i += 1;
            }
            Greater => {
                gt -= 1;
                rest.swap(i, gt);
            }
            Equal => i += 1,
        }
    }
    x.swap(0, lt); // the pivot joins the == block (x[lt] is rest[lt-1])
    (lt, gt + 1)
}

fn insertion_sort<T, F: FnMut(&T, &T) -> Ordering>(x: &mut [T], cmp: &mut F) {
    for i in 1..x.len() {
        let mut j = i;
        while j > 0 && cmp(&x[j], &x[j - 1]) == Less {
            x.swap(j, j - 1);
            j -= 1;
        }
    }
}

fn heap_sort<T, F: FnMut(&T, &T) -> Ordering>(x: &mut [T], cmp: &mut F) {
    let n = x.len();
    for i in (0..n / 2).rev() {
        sift_down(x, i, n, cmp);
    }
    for end in (1..n).rev() {
        x.swap(0, end);
        sift_down(x, 0, end, cmp);
    }
}

fn sift_down<T, F: FnMut(&T, &T) -> Ordering>(x: &mut [T], mut root: usize, n: usize, cmp: &mut F) {
    loop {
        let mut child = 2 * root + 1;
        if child >= n {
            return;
        }
        if child + 1 < n && cmp(&x[child], &x[child + 1]) == Less {
            child += 1;
        }
        if cmp(&x[root], &x[child]) != Less {
            return;
        }
        x.swap(root, child);
        root = child;
    }
}

/// Sorts x, keeping equal elements in input order. Bottom-up natural merge
/// sort: find the runs, then merge neighbours pass by pass, ping-ponging
/// between x and ONE buffer of x.len() (plus the run boundaries).
fn sort_stable<T: Clone, F: FnMut(&T, &T) -> Ordering>(x: &mut [T], mut cmp: F) {
    let n = x.len();
    if n < 2 {
        return;
    }
    let mut bounds = vec![0]; // run i is x[bounds[i]..bounds[i + 1]]
    let mut i = 0;
    while i < n {
        let mut j = i + 1;
        if j < n && cmp(&x[j], &x[i]) == Less {
            // Strictly descending: reverse it. Strict, so no two equal
            // elements trade places, and the sort stays stable.
            j += 1;
            while j < n && cmp(&x[j], &x[j - 1]) == Less {
                j += 1;
            }
            x[i..j].reverse();
        } else if j < n {
            // non-descending: that first comparison is already done
            j += 1;
            while j < n && cmp(&x[j], &x[j - 1]) != Less {
                j += 1;
            }
        }
        bounds.push(j);
        i = j;
    }
    let mut buf = x.to_vec();
    // One pass merges src into dst. A flag says where the data is: Rust will
    // not let `x` and `buf` trade places as two `&mut` (Go swaps the headers).
    let pass = |src: &[T], dst: &mut [T], bounds: &mut Vec<usize>, cmp: &mut F| {
        let mut w = 1; // rewrite bounds in place: writes trail reads
        let mut k = 0;
        while k + 1 < bounds.len() {
            let (lo, mid) = (bounds[k], bounds[k + 1]);
            let mut hi = mid;
            if k + 2 < bounds.len() {
                hi = bounds[k + 2];
                merge(&src[lo..mid], &src[mid..hi], &mut dst[lo..hi], cmp);
            } else {
                dst[lo..hi].clone_from_slice(&src[lo..hi]); // the odd run out
            }
            bounds[w] = hi;
            w += 1;
            k += 2;
        }
        bounds.truncate(w);
    };
    let mut in_buf = false;
    while bounds.len() > 2 {
        // more than one run left
        if in_buf {
            pass(&buf, x, &mut bounds, &mut cmp);
        } else {
            pass(x, &mut buf, &mut bounds, &mut cmp);
        }
        in_buf = !in_buf;
    }
    if in_buf {
        x.clone_from_slice(&buf);
    }
}

/// Merges sorted a and b into out. Ties take a: that is the stability.
fn merge<T: Clone, F: FnMut(&T, &T) -> Ordering>(a: &[T], b: &[T], out: &mut [T], cmp: &mut F) {
    let (mut i, mut j, mut k) = (0, 0, 0);
    while i < a.len() && j < b.len() {
        if cmp(&b[j], &a[i]) == Less {
            out[k] = b[j].clone();
            j += 1;
        } else {
            out[k] = a[i].clone();
            i += 1;
        }
        k += 1;
    }
    out[k..k + a.len() - i].clone_from_slice(&a[i..]);
    k += a.len() - i;
    out[k..].clone_from_slice(&b[j..]);
}

/// Reorders x so that x[k] is the element a sort would put there, with
/// x[..k] <= x[k] <= x[k+1..]. Iterative quickselect (ninther, 3-way
/// partition, insertion sort at <= 12); after bits.Len(n) bad splits the
/// remaining range is heapsorted: worst case Θ(n log n), not Θ(n²).
fn select<T, F: FnMut(&T, &T) -> Ordering>(x: &mut [T], k: usize, mut cmp: F) {
    select_counted(x, k, &mut cmp);
}

fn select_counted<T, F: FnMut(&T, &T) -> Ordering>(x: &mut [T], k: usize, cmp: &mut F) -> bool {
    assert!(k < x.len(), "select: k out of range");
    let (mut lo, mut hi) = (0, x.len());
    let mut limit = usize::BITS - x.len().leading_zeros();
    while hi - lo > 12 {
        let r = &mut x[lo..hi];
        if limit == 0 {
            heap_sort(r, cmp);
            return true;
        }
        pivot_to_front(r, cmp);
        let (lt, gt) = partition3(r, cmp);
        let len = r.len();
        if lt.max(len - gt) > len - len / 8 {
            limit -= 1;
        }
        if k - lo < lt {
            hi = lo + lt;
        } else if k - lo >= gt {
            lo += gt;
        } else {
            return false; // k landed in the == block: done
        }
    }
    insertion_sort(&mut x[lo..hi], cmp);
    false
}

// ===================== the proof, the adversary, the measurement =====================

// ---- Go's math/rand/v2 PCG (PCG-DXSM) + IntN, so the inputs are Go's inputs ----
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

#[derive(Clone, Copy, PartialEq, Debug)]
struct Rec {
    key: i64,
    tag: usize, // input position: shows stability
}

fn by_key(a: &Rec, b: &Rec) -> Ordering {
    a.key.cmp(&b.key)
}

fn by_key_tag(a: &Rec, b: &Rec) -> Ordering {
    a.key.cmp(&b.key).then(a.tag.cmp(&b.tag))
}

/// McIlroy's adversary: values are decided lazily, and whatever looks like
/// the pivot is frozen as small as possible.
struct Adversary {
    val: Vec<usize>,
    gas: usize,
    solid: usize,
    candidate: usize,
    cmps: u64,
}

impl Adversary {
    fn new(n: usize) -> Self {
        Adversary {
            val: vec![n; n],
            gas: n,
            solid: 0,
            candidate: 0,
            cmps: 0,
        }
    }
    fn cmp(&mut self, x: usize, y: usize) -> Ordering {
        self.cmps += 1;
        if self.val[x] == self.gas && self.val[y] == self.gas {
            let z = if x == self.candidate { x } else { y };
            self.val[z] = self.solid;
            self.solid += 1;
        }
        if self.val[x] == self.gas {
            self.candidate = x;
        } else if self.val[y] == self.gas {
            self.candidate = y;
        }
        self.val[x].cmp(&self.val[y])
    }
}

/// Runs one attack on the indices 0..n-1 (values exist only as the
/// adversary's answers), prints a row, returns the comparison count.
fn attack(
    name: &str,
    n: usize,
    run: impl FnOnce(&mut [usize], &mut Adversary) -> String,
    check: impl Fn(&[usize], &[usize]) -> bool,
) -> u64 {
    let mut adv = Adversary::new(n);
    let mut idx: Vec<usize> = (0..n).collect();
    let note = run(&mut idx, &mut adv);
    let nlgn = n as f64 * (n as f64).log2();
    println!(
        "   {name:<22} {:>8} comparisons = {:4.2} n lg n = {:5.2} n, {note}, correct: {}",
        adv.cmps,
        adv.cmps as f64 / nlgn,
        adv.cmps as f64 / n as f64,
        check(&idx, &adv.val)
    );
    adv.cmps
}

fn ms_once(work: &mut [i64], pristine: &[i64], f: impl FnOnce(&mut [i64])) -> f64 {
    let t = Instant::now();
    work.copy_from_slice(black_box(pristine)); // a fresh copy INSIDE the timed run
    f(work);
    black_box(&mut *work);
    t.elapsed().as_nanos() as f64 / 1e6
}

type Shape = fn(&mut Pcg, usize, usize) -> i64;
type Timed = dyn Fn(&mut [i64]);

fn main() {
    let mut r = Pcg::new(1, 2);
    const TRIALS: usize = 20000;
    println!("1. Proof against the standard library: {TRIALS} trials, n = IntN(300), 7 shapes");
    println!(
        "   (keys IntN(8), keys IntN(2^30), sorted, reversed, all-equal, organ-pipe, sawtooth)"
    );
    let shapes: [Shape; 7] = [
        |r, _, _| r.int_n(8) as i64,       // narrow: many duplicates
        |r, _, _| r.int_n(1 << 30) as i64, // wide: almost all distinct
        |_, i, _| i as i64,                // sorted
        |_, i, n| (n - 1 - i) as i64,      // reversed
        |_, _, _| 7,                       // all-equal
        |_, i, n| i.min(n - 1 - i) as i64, // organ-pipe
        |_, i, _| (i % 64) as i64,         // sawtooth
    ];
    let mut bad = [0; 4];
    let mut reordered = 0;
    for t in 0..TRIALS {
        let n = r.int_n(300);
        let input: Vec<Rec> = (0..n)
            .map(|i| Rec {
                key: shapes[t % 7](&mut r, i, n),
                tag: i,
            })
            .collect();
        // Since tags rise in input order, the stable oracle is also the input
        // sorted by (key, tag): the yardstick for "is a permutation".
        let mut want = input.clone();
        want.sort_by(by_key);
        let mut oracle = input.clone();
        oracle.sort_unstable_by(by_key);

        let mut got = input.clone();
        sort(&mut got, by_key);
        if !got.iter().zip(&oracle).all(|(a, b)| a.key == b.key) {
            bad[0] += 1;
        }
        if got != want {
            reordered += 1; // equal keys came out in a different order: legal for sort
        }
        got.sort_unstable_by(by_key_tag);
        if got != want {
            bad[1] += 1;
        }
        let mut st = input.clone();
        sort_stable(&mut st, by_key);
        if st != want {
            bad[2] += 1;
        }
        if n > 0 {
            let k = r.int_n(n);
            let mut s = input.clone();
            select(&mut s, k, by_key);
            let mut nth = input.clone();
            let (_, kth, _) = nth.select_nth_unstable_by(k, by_key); // Rust has the oracle
            let mut ok = s[k].key == kth.key;
            ok &= s[..k].iter().all(|e| e.key <= s[k].key);
            ok &= s[k + 1..].iter().all(|e| e.key >= s[k].key);
            s.sort_unstable_by(by_key_tag);
            if !ok || s != want {
                bad[3] += 1;
            }
        }
    }
    for (check, b) in [
        "sort        keys == sort_unstable_by's",
        "sort        output is a permutation of the input",
        "sort_stable == sort_by's, tags too",
        "select(k)   == select_nth_unstable_by's, left <= kth <= right",
    ]
    .iter()
    .zip(bad)
    {
        println!("   {check:<61} {b} mismatches in {TRIALS} trials");
    }
    println!(
        "   (sort reordered equal keys in {reordered} trials: it is not stable, so the tag check has teeth)"
    );

    const N: usize = 100_000;
    let nlgn = N as f64 * (N as f64).log2();
    println!("\n2. McIlroy's adversary, n = {N} (n lg n = {nlgn:.0})");
    let sorted_by = |idx: &[usize], val: &[usize]| idx.is_sorted_by_key(|&e| val[e]);
    let shipped = attack(
        "sort",
        N,
        |idx, adv| {
            let fb = sort_counted(idx, &mut |&a: &usize, &b: &usize| adv.cmp(a, b));
            format!("fallbacks {fb}")
        },
        sorted_by,
    );
    let core = attack(
        "sort, run check off",
        N,
        |idx, adv| {
            let mut fb = 0;
            let limit = usize::BITS - N.leading_zeros();
            introsort(
                idx,
                &mut |&a: &usize, &b: &usize| adv.cmp(a, b),
                limit,
                &mut fb,
            );
            format!("fallbacks {fb}")
        },
        sorted_by,
    );
    attack(
        "select(n/2)",
        N,
        |idx, adv| {
            let fell = select_counted(idx, N / 2, &mut |&a: &usize, &b: &usize| adv.cmp(a, b));
            format!("fallback {fell}")
        },
        |idx, val| {
            let kth = val[idx[N / 2]];
            idx[..N / 2].iter().all(|&e| val[e] <= kth)
                && idx[N / 2..].iter().all(|&e| val[e] >= kth)
        },
    );
    if shipped == N as u64 - 1 {
        println!(
            "   sort as shipped made n-1 comparisons: its run check goes first, the adversary's"
        );
        println!(
            "   lazy answers to it formed one ascending run, and the quicksort never started."
        );
    }
    println!(
        "   the quicksort core stays under 3 n lg n: {} (the heapsort fallback caps it)",
        core as f64 <= 3.0 * nlgn
    );

    println!("\n3. Comparison counts, n = {N}");
    let random: Vec<i64> = (0..N).map(|_| r.int_n(N) as i64).collect();
    let sorted: Vec<i64> = (0..N as i64).collect();
    let reversed: Vec<i64> = sorted.iter().rev().copied().collect();
    let equal = vec![0i64; N];
    println!(
        "   {:<11} {:>10} {:>10} {:>10} {:>10}",
        "", "random", "sorted", "reversed", "all-equal"
    );
    let mut within = true;
    for name in ["sort", "sort_stable"] {
        print!("   {name:<11}");
        for (i, xs) in [&random, &sorted, &reversed, &equal]
            .into_iter()
            .enumerate()
        {
            let mut c = 0u64;
            let counting = |a: &i64, b: &i64| {
                c += 1;
                a.cmp(b)
            };
            let mut v = xs.clone();
            if name == "sort" {
                sort(&mut v, counting);
            } else {
                sort_stable(&mut v, counting);
            }
            print!(" {c:>10}");
            within &= i == 0 || c <= 2 * N as u64;
        }
        println!();
    }
    println!("   sorted, reversed and all-equal input cost <= 2n for both: {within}");

    const BIG: usize = 1_000_000;
    println!(
        "\n4. Milliseconds per call, n = {BIG} (copy-only control subtracted; best of 3 rounds)"
    );
    let big_random: Vec<i64> = (0..BIG).map(|_| r.int_n(BIG) as i64).collect();
    let big_sorted: Vec<i64> = (0..BIG as i64).collect();
    // i64::cmp is a fn item: zero-sized, inlined into each monomorphised sort
    let rows: [(&str, &Timed); 8] = [
        ("sort", &|x| sort(x, i64::cmp)),
        ("sort_unstable_by", &|x| x.sort_unstable_by(i64::cmp)),
        ("sort_unstable", &|x| x.sort_unstable()),
        ("sort_stable", &|x| sort_stable(x, i64::cmp)),
        ("sort_by", &|x| x.sort_by(i64::cmp)),
        ("select(n/2)", &|x| select(x, BIG / 2, i64::cmp)),
        ("select_nth_unstable_by", &|x| {
            x.select_nth_unstable_by(BIG / 2, i64::cmp);
        }),
        ("sort_unstable_by, x[n/2]", &|x| {
            x.sort_unstable_by(i64::cmp);
            black_box(x[BIG / 2]);
        }),
    ];
    let inputs = [&big_random, &big_sorted];
    let mut ms = [[f64::INFINITY; 2]; 8];
    let mut ctrl = [f64::INFINITY; 2];
    let mut work = vec![0; BIG];
    for _ in 0..3 {
        // rounds interleaved: a burst of load spoils one round, not one row
        for (si, input) in inputs.iter().enumerate() {
            ctrl[si] = ctrl[si].min(ms_once(&mut work, input, |_| {}));
            for (i, (_, f)) in rows.iter().enumerate() {
                ms[i][si] = ms[i][si].min(ms_once(&mut work, input, f));
            }
        }
    }
    println!("   {:<24} {:>8} {:>8}", "", "random", "sorted");
    for (i, (name, _)) in rows.iter().enumerate() {
        ms[i][0] = (ms[i][0] - ctrl[0]).max(0.0);
        ms[i][1] = (ms[i][1] - ctrl[1]).max(0.0);
        println!("   {name:<24} {:>8.1} {:>8.1}", ms[i][0], ms[i][1]);
    }
    println!(
        "   random input: sort takes {:.2}x sort_unstable_by's time, sort_stable {:.2}x sort_by's,",
        ms[0][0] / ms[1][0],
        ms[3][0] / ms[4][0]
    );
    println!(
        "   and select(n/2) {:.2}x the time of sorting and indexing, {:.2}x select_nth_unstable_by's",
        ms[5][0] / ms[7][0],
        ms[5][0] / ms[6][0]
    );
}
```

**Rust sample output:**

```
1. Proof against the standard library: 20000 trials, n = IntN(300), 7 shapes
   (keys IntN(8), keys IntN(2^30), sorted, reversed, all-equal, organ-pipe, sawtooth)
   sort        keys == sort_unstable_by's                        0 mismatches in 20000 trials
   sort        output is a permutation of the input              0 mismatches in 20000 trials
   sort_stable == sort_by's, tags too                            0 mismatches in 20000 trials
   select(k)   == select_nth_unstable_by's, left <= kth <= right 0 mismatches in 20000 trials
   (sort reordered equal keys in 7731 trials: it is not stable, so the tag check has teeth)

2. McIlroy's adversary, n = 100000 (n lg n = 1660964)
   sort                      99999 comparisons = 0.06 n lg n =  1.00 n, fallbacks 0, correct: true
   sort, run check off     4626056 comparisons = 2.79 n lg n = 46.26 n, fallbacks 1, correct: true
   select(n/2)             4625958 comparisons = 2.79 n lg n = 46.26 n, fallback true, correct: true
   sort as shipped made n-1 comparisons: its run check goes first, the adversary's
   lazy answers to it formed one ascending run, and the quicksort never started.
   the quicksort core stays under 3 n lg n: true (the heapsort fallback caps it)

3. Comparison counts, n = 100000
                   random     sorted   reversed  all-equal
   sort           1672058      99999      99999      99999
   sort_stable    1610136      99999      99999      99999
   sorted, reversed and all-equal input cost <= 2n for both: true

4. Milliseconds per call, n = 1000000 (copy-only control subtracted; best of 3 rounds)
                              random   sorted
   sort                         64.9      0.3
   sort_unstable_by             10.1      0.2
   sort_unstable                10.1      0.2
   sort_stable                  60.9      0.9
   sort_by                      12.4      0.2
   select(n/2)                   6.3      0.7
   select_nth_unstable_by        0.9      0.8
   sort_unstable_by, x[n/2]     10.0      0.2
   random input: sort takes 6.39x sort_unstable_by's time, sort_stable 4.92x sort_by's,
   and select(n/2) 0.63x the time of sorting and indexing, 6.99x select_nth_unstable_by's
```

> **Go vs Rust:** Sections 1–3 print Go's counts exactly: 7,731 reorderings, 4,626,056 comparisons under the adversary, n − 1 on the easy shapes. The algorithms and the PCG inputs are the same, and only the oracle names change (Rust has `select_nth_unstable_by`, Go has no selection function). The timing is where the languages part. With the comparison inlined, Rust's standard library was 6–7.5× faster than this `sort` (about 10 ms against 65–80 ms) and about 4–5× faster than `sort_stable`, and `select_nth_unstable_by` found the median in about 1 ms against `select`'s 6–10 ms. So `select` beat sort-and-index by only about 1.6×, where Go's beat it by 10×. A plausible mechanism, not measured: Dijkstra's partition swaps about once per element on distinct keys, while ipnsort partitions branchlessly; in Go, the indirect `cmp` call per comparison hides that cost.

**Complexity:** `Sort` is Θ(n log n) worst case and Θ(n) on sorted, reversed and all-equal input (99,999 comparisons at n = 100,000, exactly n − 1). On random input it makes 1.007 n lg n comparisons, with O(log n) stack. `SortStable` is Θ(n log n) and Θ(n) on its natural runs, with Θ(n) extra space; `slices.SortStableFunc` uses none. `Select` is Θ(n) expected and Θ(n log n) worst case through its heapsort fallback. All four proofs held: 0 mismatches in 20,000 trials each. The tag check has teeth, because `Sort` reordered equal keys in 7,731 of those trials. McIlroy's adversary pushed the quicksort core to 2.79 n lg n (46 n), capped by one heapsort fallback. It pushed `Select(n/2)` to the same 46 n, which is the Θ(n log n) worst case that a median-of-medians fallback would remove. `Sort` as shipped never reached its quicksort: the adversary's lazy answers made the run check pass. Timed over five runs at n = 10⁶ random: `Sort` took 0.9–1.1× `slices.SortFunc`'s time, `SortStable` 0.45–0.65× `slices.SortStableFunc`'s, and `Select(n/2)` 0.1× the time of sorting and indexing (about 8.5 ms against 85–100 ms).

---

> Next lesson: [14 — Linear-Time Sorts](../../14-linear-sorts.md) — beating the n lg n bound by not
> comparing at all.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
