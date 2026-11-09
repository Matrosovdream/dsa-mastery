# Step 12 — Elementary Sorts · 🟡 Medium

Examples **7–11**: the three sorts counted on six kinds of input, insertion sort on nearly sorted
data, binary insertion sort, multi-key sorting with stable passes, and the comparator contract.

**Run any example:**

```bash
mkdir -p /tmp/dsa-ex && cd /tmp/dsa-ex   # once: go mod init scratch
go run .
```

Examples marked **Output:** are deterministic and should match your run character for character;
**Sample output:** blocks contain timings, which vary by machine.

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

## 7. Three sorts, six inputs, every operation counted

`🟡 medium` · *Comparison counting head to head*

This puts the three sorts in one table: bubble (with the early exit), selection and insertion (the shift
version), each run on six input shapes at n = 1000, with comparisons and writes counted next to the inversion
count I. You can predict every number before you look. Selection's column is flat at n(n−1)/2 comparisons with
almost no writes. Insertion's tracks I (comparisons between I and I + n − 1, shifts exactly I). Bubble pays
both, comparisons close to the maximum and 2I writes. Counting is the right tool here because the counts are
exact and the same on every machine. The cost is Θ(n²) for all three on most rows and Θ(n) for bubble and
insertion on sorted input. The one timing table then checks whether the clock agrees with a model that charges
one unit per comparison and one per write.

**Steps:**

1. Build the six inputs from one seeded PCG: sorted, reversed, a Fisher–Yates permutation, sorted with 10 random swaps, values in [0, 3), and organ pipe.
2. Run each sort on a fresh copy, record comparisons and writes, and rank the three by comparisons + writes.
3. Time the random and nearly sorted rows (copy outside the timer, best of 15) and compare the clock's ranking with the count ranking. The `ns per op` column shows where they part.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
	"time"
)

// Three elementary sorts, six input shapes, every operation counted.
//
// A "comparison" is one evaluation of < or > between two elements.
// A "write" is one slot assignment: a swap = 2 writes, an insertion-sort
// shift step = 1, dropping the held element into the hole = 1 (skipped when
// it did not move).

type stats struct{ comps, writes int }

// bubble: shrinking bound + early-exit flag (example 1).
func bubble(a []int) stats {
	var s stats
	for end := len(a) - 1; end > 0; end-- {
		swapped := false
		for j := 0; j < end; j++ {
			s.comps++
			if a[j] > a[j+1] {
				a[j], a[j+1] = a[j+1], a[j]
				s.writes += 2
				swapped = true
			}
		}
		if !swapped {
			break
		}
	}
	return s
}

// selection: scan the suffix for its first minimum, swap it to the front
// only if it is not already there (example 2).
func selection(a []int) stats {
	var s stats
	for i := 0; i < len(a)-1; i++ {
		m := i
		for j := i + 1; j < len(a); j++ {
			s.comps++
			if a[j] < a[m] {
				m = j
			}
		}
		if m != i {
			a[i], a[m] = a[m], a[i]
			s.writes += 2
		}
	}
	return s
}

// insertion: the shift (hole) version (example 3).
func insertion(a []int) stats {
	var s stats
	for i := 1; i < len(a); i++ {
		x, j := a[i], i
		for j > 0 {
			s.comps++
			if !(x < a[j-1]) {
				break
			}
			a[j] = a[j-1]
			s.writes++
			j--
		}
		if j != i {
			a[j] = x
			s.writes++
		}
	}
	return s
}

// inversions: pairs i < j with a[i] > a[j], brute force.
func inversions(a []int) int {
	inv := 0
	for i := range a {
		for j := i + 1; j < len(a); j++ {
			if a[i] > a[j] {
				inv++
			}
		}
	}
	return inv
}

type sorter struct {
	name string
	sort func([]int) stats
}

var sorters = []sorter{{"bubble", bubble}, {"selection", selection}, {"insertion", insertion}}

type input struct {
	name string
	a    []int
}

func inputs(n int, r *rand.Rand) []input {
	sorted, rev, perm, near, three, pipe := make([]int, n), make([]int, n),
		make([]int, n), make([]int, n), make([]int, n), make([]int, n)
	for i := range n {
		sorted[i], rev[i], perm[i], near[i] = i, n-1-i, i, i
		pipe[i] = min(i, n-1-i) // 0 1 2 ... 499 499 ... 2 1 0
	}
	for i := n - 1; i > 0; i-- { // Fisher-Yates on r.IntN, reproducible in Rust
		j := r.IntN(i + 1)
		perm[i], perm[j] = perm[j], perm[i]
	}
	for range 10 { // nearly sorted: 10 swaps of two random positions
		i, j := r.IntN(n), r.IntN(n)
		near[i], near[j] = near[j], near[i]
	}
	for i := range three {
		three[i] = r.IntN(3)
	}
	return []input{{"sorted", sorted}, {"reversed", rev}, {"random", perm},
		{"nearly sorted", near}, {"three values", three}, {"organ pipe", pipe}}
}

// ranking orders the sorter names by score, cheapest first.
func ranking(score []float64) string {
	idx := []int{0, 1, 2}
	slices.SortStableFunc(idx, func(x, y int) int {
		if score[x] < score[y] {
			return -1
		} else if score[x] > score[y] {
			return 1
		}
		return 0
	})
	var b strings.Builder
	for k, i := range idx {
		if k > 0 {
			if score[i] == score[idx[k-1]] {
				b.WriteString(" = ")
			} else {
				b.WriteString(" < ")
			}
		}
		b.WriteString(sorters[i].name)
	}
	return b.String()
}

// bestOf times one sort on a fresh copy of in, keeps the fastest of reps runs.
// The copy is made outside the timed region.
func bestOf(reps int, in []int, sort func([]int) stats) time.Duration {
	buf := make([]int, len(in))
	best := time.Duration(1 << 62)
	for range reps {
		copy(buf, in)
		t := time.Now()
		sort(buf)
		best = min(best, time.Since(t))
	}
	return best
}

func main() {
	const n = 1000
	r := rand.New(rand.NewPCG(1, 2))
	ins := inputs(n, r)

	fmt.Printf("n = %d; comparisons / writes (swap = 2 writes, shift step = 1, drop = 1)\n", n)
	fmt.Printf("%-14s %7s | %15s | %15s | %15s\n", "input", "I", "bubble", "selection", "insertion")
	ranks := map[string]string{}
	for _, in := range ins {
		inv := inversions(in.a)
		fmt.Printf("%-14s %7d", in.name, inv)
		ops := make([]float64, 3)
		for k, s := range sorters {
			a := slices.Clone(in.a)
			st := s.sort(a)
			if !slices.IsSorted(a) {
				panic(s.name + " did not sort")
			}
			fmt.Printf(" | %7d %7d", st.comps, st.writes)
			ops[k] = float64(st.comps + st.writes)
		}
		fmt.Println()
		ranks[in.name] = ranking(ops)
	}
	fmt.Println("\nranking by operations (comparisons + writes), cheapest first:")
	for _, in := range ins {
		fmt.Printf("  %-14s %s\n", in.name, ranks[in.name])
	}

	fmt.Println("\ntiming (best of 15 runs, copy not timed), with the operation counts beside it:")
	fmt.Printf("%-14s %-10s %10s %10s  %s\n", "input", "sort", "ops", "time µs", "ns per op")
	for _, in := range ins {
		if in.name != "random" && in.name != "nearly sorted" {
			continue
		}
		// Weighting a comparison and a write equally is a model, not a law:
		// the "ns per op" column shows what one operation really costs.
		ns, perOp := make([]float64, 3), make([]float64, 3)
		for k, s := range sorters {
			st := s.sort(slices.Clone(in.a))
			d := bestOf(15, in.a, s.sort)
			ns[k] = float64(d.Nanoseconds())
			perOp[k] = ns[k] / float64(st.comps+st.writes)
			fmt.Printf("%-14s %-10s %10d %10.1f  %.2f\n", in.name, s.name, st.comps+st.writes,
				ns[k]/1e3, perOp[k])
		}
		clock := ranking(ns)
		same := map[bool]string{true: "same as", false: "DIFFERENT from"}[clock == ranks[in.name]]
		fmt.Printf("  clock: %s  (%s the count ranking)\n", clock, same)
		fmt.Printf("  one operation costs %.1fx more in the slowest-per-op sort than in the fastest\n",
			slices.Max(perOp)/slices.Min(perOp))
	}
}
```

**Sample output:**

```
n = 1000; comparisons / writes (swap = 2 writes, shift step = 1, drop = 1)
input                I |          bubble |       selection |       insertion
sorted               0 |     999       0 |  499500       0 |     999       0
reversed        499500 |  499500  999000 |  499500    1000 |  499500  500499
random          251034 |  498720  502068 |  499500    1984 |  252027  252025
nearly sorted     8258 |  463989   16516 |  499500      20 |    9257    9125
three values    169035 |  444885  338070 |  499500    1368 |  170034  169721
organ pipe      249500 |  499500  499000 |  499500    1988 |  250499  249999

ranking by operations (comparisons + writes), cheapest first:
  sorted         bubble = insertion < selection
  reversed       selection < insertion < bubble
  random         selection < insertion < bubble
  nearly sorted  insertion < bubble < selection
  three values   insertion < selection < bubble
  organ pipe     insertion < selection < bubble

timing (best of 15 runs, copy not timed), with the operation counts beside it:
input          sort              ops    time µs  ns per op
random         bubble        1000788      412.9  0.41
random         selection      501484      292.2  0.58
random         insertion      504052       73.1  0.15
  clock: insertion < selection < bubble  (DIFFERENT from the count ranking)
  one operation costs 4.0x more in the slowest-per-op sort than in the fastest
nearly sorted  bubble         480505      200.2  0.42
nearly sorted  selection      499520      259.8  0.52
nearly sorted  insertion       18382        3.5  0.19
  clock: insertion < bubble < selection  (same as the count ranking)
  one operation costs 2.8x more in the slowest-per-op sort than in the fastest
```

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// Three elementary sorts, six input shapes, every operation counted.
//
// A "comparison" is one evaluation of < or > between two elements.
// A "write" is one slot assignment: a swap = 2 writes, an insertion-sort
// shift step = 1, dropping the held element into the hole = 1 (skipped when
// it did not move).

#[derive(Clone, Copy, Default)]
struct Stats {
    comps: u64,
    writes: u64,
}

// bubble: shrinking bound + early-exit flag (example 1).
fn bubble(a: &mut [i64]) -> Stats {
    let mut s = Stats::default();
    for end in (1..a.len()).rev() {
        let mut swapped = false;
        for j in 0..end {
            s.comps += 1;
            if a[j] > a[j + 1] {
                a.swap(j, j + 1);
                s.writes += 2;
                swapped = true;
            }
        }
        if !swapped {
            break;
        }
    }
    s
}

// selection: scan the suffix for its first minimum, swap it to the front
// only if it is not already there (example 2). Note for the timing table:
// rustc -O compiles `if a[j] < a[m] { m = j }` to a branchless select (csel,
// seen in the assembly), so every step's load of a[m] waits for the previous
// step's select. A branch would be predicted (the minimum rarely changes);
// keeping the minimum's value in a local would also break the chain.
fn selection(a: &mut [i64]) -> Stats {
    let mut s = Stats::default();
    for i in 0..a.len().saturating_sub(1) {
        let mut m = i;
        for j in i + 1..a.len() {
            s.comps += 1;
            if a[j] < a[m] {
                m = j;
            }
        }
        if m != i {
            a.swap(i, m);
            s.writes += 2;
        }
    }
    s
}

// insertion: the shift (hole) version (example 3). i64 is Copy, so holding
// x = a[i] leaves a[i] readable and the "hole" is just a slot we overwrite.
fn insertion(a: &mut [i64]) -> Stats {
    let mut s = Stats::default();
    for i in 1..a.len() {
        let (x, mut j) = (a[i], i);
        while j > 0 {
            s.comps += 1;
            if x >= a[j - 1] {
                break;
            }
            a[j] = a[j - 1];
            s.writes += 1;
            j -= 1;
        }
        if j != i {
            a[j] = x;
            s.writes += 1;
        }
    }
    s
}

// inversions: pairs i < j with a[i] > a[j], brute force.
fn inversions(a: &[i64]) -> u64 {
    a.iter()
        .enumerate()
        .map(|(i, x)| a[i + 1..].iter().filter(|y| x > y).count() as u64)
        .sum()
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so the
// random inputs are exactly the ones the Go program sorts.
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

type SortFn = fn(&mut [i64]) -> Stats;
const SORTERS: [(&str, SortFn); 3] = [
    ("bubble", bubble),
    ("selection", selection),
    ("insertion", insertion),
];

fn inputs(n: usize, r: &mut Pcg) -> Vec<(&'static str, Vec<i64>)> {
    let sorted: Vec<i64> = (0..n as i64).collect();
    let rev: Vec<i64> = sorted.iter().rev().copied().collect();
    let pipe: Vec<i64> = (0..n).map(|i| i.min(n - 1 - i) as i64).collect();
    let mut perm = sorted.clone();
    for i in (1..n).rev() {
        // Fisher-Yates on int_n, the same calls in the same order as Go
        let j = r.int_n(i + 1);
        perm.swap(i, j);
    }
    let mut near = sorted.clone();
    for _ in 0..10 {
        // nearly sorted: 10 swaps of two random positions
        let (i, j) = (r.int_n(n), r.int_n(n));
        near.swap(i, j);
    }
    let three: Vec<i64> = (0..n).map(|_| r.int_n(3) as i64).collect();
    vec![
        ("sorted", sorted),
        ("reversed", rev),
        ("random", perm),
        ("nearly sorted", near),
        ("three values", three),
        ("organ pipe", pipe),
    ]
}

/// ranking orders the sorter names by score, cheapest first ("=" for a tie).
fn ranking(score: &[f64; 3]) -> String {
    let mut idx = [0, 1, 2];
    idx.sort_by(|&x, &y| score[x].total_cmp(&score[y])); // stable, like Go's
    let mut out = String::from(SORTERS[idx[0]].0);
    for w in idx.windows(2) {
        out += if score[w[1]] == score[w[0]] {
            " = "
        } else {
            " < "
        };
        out += SORTERS[w[1]].0;
    }
    out
}

/// best_of times one sort on a fresh copy of `input`, keeps the fastest of
/// `reps` runs. The copy is made outside the timed region.
fn best_of(reps: usize, input: &[i64], sort: SortFn) -> Duration {
    let mut buf = vec![0; input.len()];
    let mut best = Duration::MAX;
    for _ in 0..reps {
        buf.copy_from_slice(input);
        let t = Instant::now();
        black_box(sort(black_box(&mut buf)));
        best = best.min(t.elapsed());
    }
    best
}

fn main() {
    const N: usize = 1000;
    let mut r = Pcg::new(1, 2);
    let ins = inputs(N, &mut r);

    println!("n = {N}; comparisons / writes (swap = 2 writes, shift step = 1, drop = 1)");
    println!(
        "{:<14} {:>7} | {:>15} | {:>15} | {:>15}",
        "input", "I", "bubble", "selection", "insertion"
    );
    let mut ranks = Vec::new();
    for (name, input) in &ins {
        let mut line = format!("{:<14} {:>7}", name, inversions(input));
        let mut ops = [0.0; 3];
        for (k, (sname, sort)) in SORTERS.iter().enumerate() {
            let mut a = input.clone();
            let st = sort(&mut a);
            assert!(a.is_sorted(), "{sname} did not sort");
            line += &format!(" | {:>7} {:>7}", st.comps, st.writes);
            ops[k] = (st.comps + st.writes) as f64;
        }
        println!("{line}");
        ranks.push(ranking(&ops));
    }
    println!("\nranking by operations (comparisons + writes), cheapest first:");
    for ((name, _), rank) in ins.iter().zip(&ranks) {
        println!("  {name:<14} {rank}");
    }

    println!("\ntiming (best of 15 runs, copy not timed), with the operation counts beside it:");
    println!(
        "{:<14} {:<10} {:>10} {:>10}  ns per op",
        "input", "sort", "ops", "time µs"
    );
    for ((name, input), rank) in ins.iter().zip(&ranks) {
        if *name != "random" && *name != "nearly sorted" {
            continue;
        }
        // Weighting a comparison and a write equally is a model, not a law:
        // the "ns per op" column shows what one operation really costs.
        let (mut ns, mut per_op) = ([0.0; 3], [0.0f64; 3]);
        for (k, (sname, sort)) in SORTERS.iter().enumerate() {
            let st = sort(&mut input.clone());
            let ops = st.comps + st.writes;
            ns[k] = best_of(15, input, *sort).as_nanos() as f64;
            per_op[k] = ns[k] / ops as f64;
            println!(
                "{:<14} {:<10} {:>10} {:>10.1}  {:.2}",
                name,
                sname,
                ops,
                ns[k] / 1e3,
                per_op[k]
            );
        }
        let clock = ranking(&ns);
        let same = if clock == *rank {
            "same as"
        } else {
            "DIFFERENT from"
        };
        println!("  clock: {clock}  ({same} the count ranking)");
        let (lo, hi) = per_op
            .iter()
            .fold((f64::MAX, 0.0f64), |(lo, hi), &x| (lo.min(x), hi.max(x)));
        println!(
            "  one operation costs {:.1}x more in the slowest-per-op sort than in the fastest",
            hi / lo
        );
    }
}
```

**Rust sample output:**

```
n = 1000; comparisons / writes (swap = 2 writes, shift step = 1, drop = 1)
input                I |          bubble |       selection |       insertion
sorted               0 |     999       0 |  499500       0 |     999       0
reversed        499500 |  499500  999000 |  499500    1000 |  499500  500499
random          251034 |  498720  502068 |  499500    1984 |  252027  252025
nearly sorted     8258 |  463989   16516 |  499500      20 |    9257    9125
three values    169035 |  444885  338070 |  499500    1368 |  170034  169721
organ pipe      249500 |  499500  499000 |  499500    1988 |  250499  249999

ranking by operations (comparisons + writes), cheapest first:
  sorted         bubble = insertion < selection
  reversed       selection < insertion < bubble
  random         selection < insertion < bubble
  nearly sorted  insertion < bubble < selection
  three values   insertion < selection < bubble
  organ pipe     insertion < selection < bubble

timing (best of 15 runs, copy not timed), with the operation counts beside it:
input          sort              ops    time µs  ns per op
random         bubble        1000788      473.8  0.47
random         selection      501484      857.8  1.71
random         insertion      504052       72.5  0.14
  clock: insertion < bubble < selection  (DIFFERENT from the count ranking)
  one operation costs 11.9x more in the slowest-per-op sort than in the fastest
nearly sorted  bubble         480505      262.4  0.55
nearly sorted  selection      499520      809.0  1.62
nearly sorted  insertion       18382        3.1  0.17
  clock: insertion < bubble < selection  (same as the count ranking)
  one operation costs 9.5x more in the slowest-per-op sort than in the fastest
```

> **Go vs Rust:** The count table and the count ranking are identical to Go's line for line, because `Pcg` ports `math/rand/v2` and the three sorts ask the same questions in the same order. The clock is not. Rust's selection sort costs about 1.7 ns per operation against Go's 0.6, which drops it behind bubble on the random row (Rust: insertion < bubble < selection; Go: insertion < selection < bubble). From reading the assembly, not from a measurement of the cause: rustc compiles `if a[j] < a[m] { m = j }` to a `csel`, so each step's load of `a[m]` waits for the previous step, while Go emits a branch that is almost always predicted. Insertion sort costs the same in both, about 0.14 ns per operation on the random row.

**Complexity:** selection makes n(n−1)/2 = 499,500 comparisons on every row, with at most n−1 swaps · insertion makes I to I + n − 1 comparisons and exactly I shifts (252,027 comparisons on the random row, 9,257 on the nearly sorted row) · bubble's swaps are I as well, so it makes 2I writes, and its comparisons stay near n(n−1)/2 unless the input is sorted · on sorted input bubble and insertion make n − 1 = 999 comparisons, Θ(n). On random input selection and insertion do about the same number of operations, yet insertion runs several times faster: equal weights for a comparison and a write predict which sort does the least work, but not which one finishes first.

---

## 8. Nearly sorted input: where insertion sort wins

`🟡 medium` · *Adaptive, O(nk)*

Call a slice k-sorted when every element is at most k places from where it belongs. Each element then crosses
at most k others, so the slice has fewer than n·k inversions, and insertion sort, which costs Θ(n + I), runs in
O(nk). That is linear in n for any fixed k. `slices.Sort` is Θ(n log n) on such input because pdqsort's
presorted shortcut gives up after 5 fixes. This is the case for calling insertion sort on a *large* input, the
opposite of [lesson 01 example 11](../01-introduction/3-hard.md#11-big-o-lies-at-small-n), where it only won at
small n. Here n = 1,000,000 and k grows from 0 to 1024. Comparison counts show the shape, one timing table shows
where the shape stops paying off, and k = 256 and 1024 are in the table to find the k where it stops.

**Steps:**

1. Build each k-sorted input: 0..n−1, then a Fisher–Yates shuffle inside each disjoint block of k+1 slots (seeded PCG). Count I block by block, since no inversion crosses a block boundary.
2. Count comparisons for insertion sort and, with a counting closure, for `slices.SortFunc` and `slices.SortStableFunc`. Check shifts = I and I ≤ comparisons ≤ I + n − 1.
3. Time all three per k (copy outside the timer, best of 5) and find the k where insertion stops winning. Then try the other common nearly sorted case: a sorted slice with 10 random values appended.

```go
package main

import (
	"cmp"
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
	"time"
)

// A slice is "k-sorted" when every element is at most k places from where
// it belongs. An element k places from home is part of at most k inversions
// with the elements it must cross, so the total inversion count is I < n*k
// and insertion sort -- Theta(n + I) -- runs in O(n*k): linear for fixed k,
// however large n is. A general O(n log n) sort has no such guarantee.

// insertion is the plain shift version, used for timing.
func insertion(a []int) {
	for i := 1; i < len(a); i++ {
		x, j := a[i], i
		for j > 0 && x < a[j-1] {
			a[j] = a[j-1]
			j--
		}
		a[j] = x
	}
}

// insertionCounted is the same algorithm, counting comparisons and shifts.
func insertionCounted(a []int) (comps, shifts int) {
	for i := 1; i < len(a); i++ {
		x, j := a[i], i
		for j > 0 {
			comps++
			if !(x < a[j-1]) {
				break
			}
			a[j] = a[j-1]
			shifts++
			j--
		}
		a[j] = x
	}
	return
}

// kSorted: 0..n-1, then a Fisher-Yates shuffle inside each disjoint block of
// k+1 slots. Nothing leaves its block, so nothing moves more than k places.
// It also returns I, counted block by block (inversions never cross a block
// boundary here), which is O(n*k) instead of the O(n^2) brute force.
func kSorted(n, k int, r *rand.Rand) (a []int, inv int) {
	a = make([]int, n)
	for i := range a {
		a[i] = i
	}
	for lo := 0; lo < n; lo += k + 1 {
		b := a[lo:min(lo+k+1, n)]
		for i := len(b) - 1; i > 0; i-- {
			j := r.IntN(i + 1)
			b[i], b[j] = b[j], b[i]
		}
		for i := range b {
			for j := i + 1; j < len(b); j++ {
				if b[i] > b[j] {
					inv++
				}
			}
		}
	}
	return a, inv
}

func maxDisplacement(a []int) int { // a is a permutation of 0..n-1
	d := 0
	for i, v := range a {
		d = max(d, v-i, i-v)
	}
	return d
}

// stdComps counts the comparator calls a stdlib sort makes on a copy of a.
func stdComps(a []int, sort func([]int, func(x, y int) int)) int {
	c := 0
	sort(slices.Clone(a), func(x, y int) int { c++; return cmp.Compare(x, y) })
	return c
}

// bestOf times sort on a fresh copy of in (copy not timed), fastest of reps.
func bestOf(reps int, in []int, sort func([]int)) time.Duration {
	buf := make([]int, len(in))
	best := time.Duration(1 << 62)
	for range reps {
		copy(buf, in)
		t := time.Now()
		sort(buf)
		best = min(best, time.Since(t))
		if !slices.IsSorted(buf) {
			panic("not sorted")
		}
	}
	return best
}

func ms(d time.Duration) float64 { return float64(d.Nanoseconds()) / 1e6 }

func main() {
	const n = 1_000_000
	ks := []int{0, 1, 2, 4, 8, 16, 64, 256, 1024} // 256 and 1024: to find where it stops winning
	r := rand.New(rand.NewPCG(1, 2))
	inputs := make([][]int, len(ks))

	fmt.Printf("n = %d, k-sorted input: comparisons per sort\n", n)
	fmt.Printf("%4s %6s %10s | %11s %11s | %10s %14s\n",
		"k", "max d", "I", "insertion", "/ n", "SortFunc", "SortStableFunc")
	identities := true
	for t, k := range ks {
		a, inv := kSorted(n, k, r)
		inputs[t] = a
		c, sh := insertionCounted(slices.Clone(a))
		identities = identities && sh == inv && inv <= c && c <= inv+n-1
		fmt.Printf("%4d %6d %10d | %11d %11.2f | %10d %14d\n", k, maxDisplacement(a), inv, c,
			float64(c)/n, stdComps(a, slices.SortFunc[[]int]), stdComps(a, slices.SortStableFunc[[]int]))
	}
	fmt.Printf("insertion: shifts == I and I <= comparisons <= I+n-1 on every row: %v\n", identities)

	fmt.Println("\ntiming, ms (best of 5, copy not timed)")
	fmt.Printf("%4s %10s %11s %15s   %s\n", "k", "insertion", "slices.Sort", "SortStableFunc", "insertion vs the faster stdlib sort")
	var wins []string
	for t, k := range ks {
		ti := bestOf(5, inputs[t], insertion)
		ts := bestOf(5, inputs[t], slices.Sort[[]int])
		tst := bestOf(5, inputs[t], func(a []int) { slices.SortStableFunc(a, cmp.Compare[int]) })
		ratio := float64(min(ts, tst)) / float64(ti) // > 1: insertion is faster
		verdict := "within 10%: a tie"
		if ratio > 1.1 {
			verdict = "insertion faster"
			wins = append(wins, fmt.Sprint(k))
		} else if ratio < 1/1.1 {
			verdict = "insertion slower"
		}
		fmt.Printf("%4d %10.2f %11.2f %15.2f   %.2fx  %s\n", k, ms(ti), ms(ts), ms(tst), ratio, verdict)
	}
	fmt.Printf("insertion sort beat both stdlib sorts by more than 10%% at k = %s\n", strings.Join(wins, ", "))

	// A sorted slice with a few late arrivals appended: each one walks left
	// past every larger element, so the cost is the distance they travel.
	app := make([]int, n, n+10)
	for i := range app {
		app[i] = i
	}
	for range 10 {
		app = append(app, r.IntN(n))
	}
	c, sh := insertionCounted(slices.Clone(app))
	ti, ts := bestOf(5, app, insertion), bestOf(5, app, slices.Sort[[]int])
	fmt.Printf("\nsorted %d + 10 random appended: insertion %d comparisons (%d shifts), %.2f ms; slices.Sort %.2f ms\n",
		n, c, sh, ms(ti), ms(ts))
}
```

**Sample output:**

```
n = 1000000, k-sorted input: comparisons per sort
   k  max d          I |   insertion         / n |   SortFunc SortStableFunc
   0      0          0 |      999999        1.00 |    1000011        1200386
   1      1     249858 |     1249856        1.25 |   19374936        1425258
   2      2     498958 |     1498957        1.50 |   19296423        1887826
   4      4    1000833 |     2000831        2.00 |   19191061        2136977
   8      8    1999185 |     2999183        3.00 |   19377898        3379590
  16     16    4000125 |     5000121        5.00 |   19579756        4860831
  64     64   15982673 |    16982666       16.98 |   19941969        7642113
 256    256   63960993 |    64960986       64.96 |   20169270       10145741
1024   1023  256125711 |   257125707      257.13 |   20374901       12589348
insertion: shifts == I and I <= comparisons <= I+n-1 on every row: true

timing, ms (best of 5, copy not timed)
   k  insertion slices.Sort  SortStableFunc   insertion vs the faster stdlib sort
   0       0.51        0.51            2.12   1.01x  within 10%: a tie
   1       2.15       11.39            4.33   2.01x  insertion faster
   2       3.27       13.29            6.58   2.01x  insertion faster
   4       4.55       15.80            8.04   1.77x  insertion faster
   8       6.01       18.49           13.06   2.17x  insertion faster
  16       7.08       20.08           17.92   2.53x  insertion faster
  64      10.67       24.43           31.18   2.29x  insertion faster
 256      24.71       29.38           49.18   1.19x  insertion faster
1024      76.77       34.21           67.90   0.45x  insertion slower
insertion sort beat both stdlib sorts by more than 10% at k = 1, 2, 4, 8, 16, 64, 256

sorted 1000000 + 10 random appended: insertion 5718474 comparisons (4718465 shifts), 1.76 ms; slices.Sort 5.06 ms
```

**Rust:**

```rust
use std::cmp::Ordering;
use std::hint::black_box;
use std::time::{Duration, Instant};

// A slice is "k-sorted" when every element is at most k places from where
// it belongs. An element k places from home is part of at most k inversions
// with the elements it must cross, so the total inversion count is I < n*k
// and insertion sort -- Theta(n + I) -- runs in O(n*k): linear for fixed k,
// however large n is. A general O(n log n) sort has no such guarantee.

// insertion is the plain shift version, used for timing. i64 is Copy, so
// holding x = a[i] and overwriting a[i] later is ordinary safe code.
fn insertion(a: &mut [i64]) {
    for i in 1..a.len() {
        let (x, mut j) = (a[i], i);
        while j > 0 && x < a[j - 1] {
            a[j] = a[j - 1];
            j -= 1;
        }
        a[j] = x;
    }
}

// insertion_counted is the same algorithm, counting comparisons and shifts.
fn insertion_counted(a: &mut [i64]) -> (u64, u64) {
    let (mut comps, mut shifts) = (0, 0);
    for i in 1..a.len() {
        let (x, mut j) = (a[i], i);
        while j > 0 {
            comps += 1;
            if x >= a[j - 1] {
                break;
            }
            a[j] = a[j - 1];
            shifts += 1;
            j -= 1;
        }
        a[j] = x;
    }
    (comps, shifts)
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so every
// input below is exactly the one the Go program sorts.
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

// k_sorted: 0..n-1, then a Fisher-Yates shuffle inside each disjoint block
// of k+1 slots (chunks_mut). Nothing leaves its block, so nothing moves more
// than k places. It also returns I, counted block by block (inversions never
// cross a block boundary here): O(n*k) instead of the O(n^2) brute force.
fn k_sorted(n: usize, k: usize, r: &mut Pcg) -> (Vec<i64>, u64) {
    let mut a: Vec<i64> = (0..n as i64).collect();
    let mut inv = 0;
    for b in a.chunks_mut(k + 1) {
        for i in (1..b.len()).rev() {
            let j = r.int_n(i + 1);
            b.swap(i, j);
        }
        for (i, x) in b.iter().enumerate() {
            inv += b[i + 1..].iter().filter(|y| x > y).count() as u64;
        }
    }
    (a, inv)
}

fn max_displacement(a: &[i64]) -> u64 {
    // a is a permutation of 0..n-1
    a.iter()
        .enumerate()
        .map(|(i, &v)| (i as i64).abs_diff(v))
        .max()
        .unwrap_or(0)
}

// std_comps counts the comparator calls a std sort makes on a copy of a.
type Cmp<'a> = &'a mut dyn FnMut(&i64, &i64) -> Ordering;
fn std_comps(a: &[i64], sort: fn(&mut [i64], Cmp)) -> u64 {
    let mut c = 0;
    sort(&mut a.to_vec(), &mut |x, y| {
        c += 1;
        x.cmp(y)
    });
    c
}

// best_of times sort on a fresh copy of input (copy not timed), fastest of reps.
fn best_of(reps: usize, input: &[i64], sort: impl Fn(&mut [i64])) -> Duration {
    let mut buf = vec![0; input.len()];
    let mut best = Duration::MAX;
    for _ in 0..reps {
        buf.copy_from_slice(input);
        let t = Instant::now();
        sort(black_box(&mut buf));
        best = best.min(t.elapsed());
        assert!(buf.is_sorted(), "not sorted");
    }
    best
}

fn ms(d: Duration) -> f64 {
    d.as_nanos() as f64 / 1e6
}

fn main() {
    const N: usize = 1_000_000;
    let ks = [0, 1, 2, 4, 8, 16, 64, 256, 1024]; // 256 and 1024: to find where it stops winning
    let mut r = Pcg::new(1, 2);
    let mut inputs = Vec::new();

    println!("n = {N}, k-sorted input: comparisons per sort");
    println!(
        "{:>4} {:>6} {:>10} | {:>11} {:>11} | {:>16} {:>10}",
        "k", "max d", "I", "insertion", "/ n", "sort_unstable_by", "sort_by"
    );
    let mut identities = true;
    for &k in &ks {
        let (a, inv) = k_sorted(N, k, &mut r);
        let (c, sh) = insertion_counted(&mut a.clone());
        identities &= sh == inv && inv <= c && c < inv + N as u64; // c <= I + n - 1
        println!(
            "{:>4} {:>6} {:>10} | {:>11} {:>11.2} | {:>16} {:>10}",
            k,
            max_displacement(&a),
            inv,
            c,
            c as f64 / N as f64,
            std_comps(&a, |v, f| v.sort_unstable_by(f)),
            std_comps(&a, |v, f| v.sort_by(f))
        );
        inputs.push(a);
    }
    println!("insertion: shifts == I and I <= comparisons <= I+n-1 on every row: {identities}");

    println!("\ntiming, ms (best of 5, copy not timed)");
    println!(
        "{:>4} {:>10} {:>13} {:>8}   insertion vs the faster std sort",
        "k", "insertion", "sort_unstable", "sort"
    );
    let mut wins = Vec::new();
    for (&k, a) in ks.iter().zip(&inputs) {
        let ti = best_of(5, a, insertion);
        let tu = best_of(5, a, |v| v.sort_unstable());
        let ts = best_of(5, a, |v| v.sort());
        let ratio = tu.min(ts).as_secs_f64() / ti.as_secs_f64(); // > 1: insertion is faster
        let verdict = if ratio > 1.1 {
            wins.push(k.to_string());
            "insertion faster"
        } else if ratio < 1.0 / 1.1 {
            "insertion slower"
        } else {
            "within 10%: a tie"
        };
        println!(
            "{:>4} {:>10.2} {:>13.2} {:>8.2}   {:.2}x  {}",
            k,
            ms(ti),
            ms(tu),
            ms(ts),
            ratio,
            verdict
        );
    }
    println!(
        "insertion sort beat both std sorts by more than 10% at k = {}",
        wins.join(", ")
    );

    // A sorted slice with a few late arrivals appended: each one walks left
    // past every larger element, so the cost is the distance they travel.
    let mut app: Vec<i64> = Vec::with_capacity(N + 10);
    app.extend(0..N as i64);
    for _ in 0..10 {
        app.push(r.int_n(N) as i64);
    }
    let (c, sh) = insertion_counted(&mut app.clone());
    let (ti, tu) = (
        best_of(5, &app, insertion),
        best_of(5, &app, |v| v.sort_unstable()),
    );
    println!(
        "\nsorted {N} + 10 random appended: insertion {c} comparisons ({sh} shifts), {:.2} ms; sort_unstable {:.2} ms",
        ms(ti),
        ms(tu)
    );
}
```

**Rust sample output:**

```
n = 1000000, k-sorted input: comparisons per sort
   k  max d          I |   insertion         / n | sort_unstable_by    sort_by
   0      0          0 |      999999        1.00 |           999999     999999
   1      1     249858 |     1249856        1.25 |         20395134   18928289
   2      2     498958 |     1498957        1.50 |         20381185   19023025
   4      4    1000833 |     2000831        2.00 |         20420065   19171328
   8      8    1999185 |     2999183        3.00 |         20524184   19472821
  16     16    4000125 |     5000121        5.00 |         20564887   19942658
  64     64   15982673 |    16982666       16.98 |         20508311   20473750
 256    256   63960993 |    64960986       64.96 |         20588019   20757149
1024   1023  256125711 |   257125707      257.13 |         20699096   20922571
insertion: shifts == I and I <= comparisons <= I+n-1 on every row: true

timing, ms (best of 5, copy not timed)
   k  insertion sort_unstable     sort   insertion vs the faster std sort
   0       0.51          0.26     0.27   0.50x  insertion slower
   1       2.06          8.43     9.01   4.09x  insertion faster
   2       2.97          8.99     9.69   3.02x  insertion faster
   4       4.48          9.24    10.42   2.07x  insertion faster
   8       5.66          9.47    11.15   1.67x  insertion faster
  16       6.57          9.50    11.58   1.45x  insertion faster
  64      10.26          9.55    12.18   0.93x  within 10%: a tie
 256      24.83          9.80    12.27   0.39x  insertion slower
1024      80.67         11.13    12.71   0.14x  insertion slower
insertion sort beat both std sorts by more than 10% at k = 1, 2, 4, 8, 16

sorted 1000000 + 10 random appended: insertion 5718474 comparisons (4718465 shifts), 1.75 ms; sort_unstable 8.44 ms
```

> **Go vs Rust:** The first five columns (k, max d, I, insertion comparisons, / n) and the counts on the appended-values line match Go exactly. The std columns differ by design. `sort_by` (driftsort) makes about 19–21 million comparisons for every k ≥ 1, where Go's `SortStableFunc` makes 1.4–12.6 million. Driftsort keeps an existing ascending run only if it is at least `min_good_run_len` ≈ √n long (`core/src/slice/sort/stable/drift.rs`), and k-sorted input has no such run. Go's stable sort starts by insertion-sorting blocks of 20, which adapts. Both Rust sorts make n − 1 comparisons at k = 0 (`find_existing_run`).
> Rust's std sorts are faster than Go's on these inputs: `sort_unstable` takes about 9–10 ms at every k ≥ 1, against 11–35 ms for `slices.Sort`. So insertion sort's lead ends sooner: about even at k = 64 and behind from k = 256 (in Go it still wins at 256). At k = 0 the std run scan is about twice as fast as insertion sort, which also writes every element back.

**Complexity:** insertion sort makes I + n − 1 comparisons or fewer, with I < n·k, so it is O(nk): about 1.25n comparisons at k = 1, 17n at k = 64 and 257n at k = 1024 · `slices.SortFunc` makes about 19–20 million comparisons (≈ n log₂ n) for every k ≥ 1, and only n + 12 on sorted input · `slices.SortStableFunc` adapts partly, since it insertion-sorts blocks of 20 before merging (1.4 million comparisons at k = 1, 12.6 million at k = 1024) · measured here, insertion sort beats both stdlib sorts by about 2× from k = 1 to k = 64, by about 1.2× at k = 256, and is about 2× slower at k = 1024. Its cost grows with k, the distance from sorted, while pdqsort's stays at about n log₂ n whatever k is, so k decides the crossover · 10 values appended to a sorted million: about 5.7 million comparisons, roughly 2 ms against about 5 ms for `slices.Sort`.

---

## 9. Binary insertion sort: fewer comparisons, same moves

`🟡 medium` · *Comparisons vs moves*

The sorted prefix invites a binary search: find the slot for `s[i]` in about log₂ i comparisons, then move the
block between the slot and `i` one place right with a single `copy`. Comparisons drop from Θ(n²) to Θ(n log n),
here 114× fewer at n = 5000. The moves do not drop at all. Every element still travels exactly as far as before,
so the shifts equal I and stay Θ(n²) on average. The trade pays when a comparison costs much more than a move,
as with strings that share a long prefix. The program times ints and 46-byte strings, with a control that keeps
the linear search but moves by block, so you can see how much each half of the change is worth. There is also a
stability trap. The search must return the **upper** bound, the slot after the last equal key.
`slices.BinarySearchFunc` returns the earliest position, which inserts before the equal keys and quietly reverses
their order.

**Steps:**

1. Write the linear shift version, a scan+copy control (linear search, one block `copy`) and binary insertion, all generic over `cmp` and counting comparisons and shifts.
2. Sort one PCG permutation of n = 5000, first as ints, then as strings with a 40-byte common prefix. Check shifts = I, then time the three versions.
3. Sort 16 records with keys in [0, 4) using a lower-bound search (`slices.BinarySearchFunc`) and an upper-bound search (`sort.Search` for the first element > x), and check each for stability.

```go
package main

import (
	"cmp"
	"fmt"
	"math/rand/v2"
	"slices"
	"sort"
	"strings"
	"time"
)

// Binary insertion sort: the prefix s[:i] is sorted, so find the slot for
// s[i] by binary search -- about log2(i) comparisons instead of up to i --
// then move the elements in between one place right. The search makes the
// comparisons Theta(n log n), but the move is the same distance as before:
// shifts are still exactly I, the inversion count, Theta(n^2) on average.

type counts struct{ comps, shifts int }

// linear is the ordinary shift version.
func linear[E any](s []E, cmp func(a, b E) int) (c counts) {
	for i := 1; i < len(s); i++ {
		x, j := s[i], i
		for j > 0 {
			c.comps++
			if cmp(x, s[j-1]) >= 0 { // stop at the first element <= x: stable
				break
			}
			s[j] = s[j-1]
			c.shifts++
			j--
		}
		s[j] = x
	}
	return c
}

// scanCopy is a control for the timing table: the SAME linear search (same
// comparisons as linear), but the elements are moved afterwards by one copy
// instead of one at a time. It separates "fewer comparisons" from "block move".
func scanCopy[E any](s []E, cmp func(a, b E) int) (c counts) {
	for i := 1; i < len(s); i++ {
		x, j := s[i], i
		for j > 0 {
			c.comps++
			if cmp(x, s[j-1]) >= 0 {
				break
			}
			j--
		}
		copy(s[j+1:i+1], s[j:i])
		s[j] = x
		c.shifts += i - j
	}
	return c
}

// binary finds the slot by bisection. upper = true: insert AFTER the last
// element equal to x (sort.Search for the first element > x) -- stable.
// upper = false: slices.BinarySearchFunc, which returns the EARLIEST
// position, i.e. BEFORE the equal elements -- sorted, but not stable.
func binary[E any](s []E, cmp func(a, b E) int, upper bool) (c counts) {
	count := func(a, b E) int { c.comps++; return cmp(a, b) }
	for i := 1; i < len(s); i++ {
		x := s[i]
		var pos int
		if upper {
			pos = sort.Search(i, func(j int) bool { return count(s[j], x) > 0 })
		} else {
			pos, _ = slices.BinarySearchFunc(s[:i], x, count)
		}
		copy(s[pos+1:i+1], s[pos:i]) // one block move of i-pos elements
		s[pos] = x
		c.shifts += i - pos
	}
	return c
}

func inversions(a []int) int {
	inv := 0
	for i := range a {
		for j := i + 1; j < len(a); j++ {
			if a[i] > a[j] {
				inv++
			}
		}
	}
	return inv
}

// bestOf times sort on a fresh copy of in (copy not timed), fastest of reps.
func bestOf[E any](reps int, in []E, sort func([]E) counts) time.Duration {
	buf := make([]E, len(in))
	best := time.Duration(1 << 62)
	for range reps {
		copy(buf, in)
		t := time.Now()
		sort(buf)
		best = min(best, time.Since(t))
	}
	return best
}

type rec struct{ key, tag int }

func cmpKey(a, b rec) int { return cmp.Compare(a.key, b.key) }

func show(rs []rec) string {
	parts := make([]string, len(rs))
	for i, r := range rs {
		parts[i] = fmt.Sprintf("%d.%d", r.key, r.tag)
	}
	return strings.Join(parts, " ")
}

// firstViolation: the first adjacent pair with equal keys in the wrong tag order.
func firstViolation(rs []rec) string {
	for i := 1; i < len(rs); i++ {
		if rs[i-1].key == rs[i].key && rs[i-1].tag > rs[i].tag {
			return fmt.Sprintf("at %d: %d.%d before %d.%d", i-1, rs[i-1].key, rs[i-1].tag, rs[i].key, rs[i].tag)
		}
	}
	return "none"
}

func main() {
	const n = 5000
	r := rand.New(rand.NewPCG(1, 2))
	ints := make([]int, n)
	for i := range ints {
		ints[i] = i
	}
	for i := n - 1; i > 0; i-- {
		j := r.IntN(i + 1)
		ints[i], ints[j] = ints[j], ints[i]
	}
	inv := inversions(ints)

	fmt.Printf("(a) n = %d random permutation, I = %d\n", n, inv)
	a, b := slices.Clone(ints), slices.Clone(ints)
	cl, cb := linear(a, cmp.Compare[int]), binary(b, cmp.Compare[int], true)
	fmt.Printf("  linear: %8d comparisons, %8d shifts\n", cl.comps, cl.shifts)
	fmt.Printf("  binary: %8d comparisons, %8d shifts\n", cb.comps, cb.shifts)
	fmt.Printf("  shifts == I for both: %v; binary makes %.0fx fewer comparisons; both sorted: %v\n",
		cl.shifts == inv && cb.shifts == inv, float64(cl.comps)/float64(cb.comps),
		slices.IsSorted(a) && slices.IsSorted(b))

	// The same permutation as strings that share a 40-byte prefix: the same
	// order, so the same counts, but every comparison now walks 40+ bytes.
	strs := make([]string, n)
	for i, v := range ints {
		strs[i] = fmt.Sprintf("customer-account-region-eu-west-0000000-%06d", v)
	}
	cs := linear(slices.Clone(strs), strings.Compare)
	fmt.Printf("\n(b) the same permutation as %d-byte strings with a %d-byte common prefix\n",
		len(strs[0]), len("customer-account-region-eu-west-0000000-"))
	fmt.Printf("  string counts equal int counts (same order): %v\n", cs == cl)

	fmt.Printf("  scan+copy (linear search, one block copy) counts equal linear's: %v\n",
		scanCopy(slices.Clone(strs), strings.Compare) == cl)

	fmt.Println("  timing in ms, best of 5, copy of the input not timed:")
	fmt.Printf("  %-8s %10s %10s %10s\n", "", "linear", "scan+copy", "binary")
	ms := func(d time.Duration) float64 { return float64(d.Nanoseconds()) / 1e6 }
	var lin, sc, bin [2]time.Duration
	lin[0] = bestOf(5, ints, func(s []int) counts { return linear(s, cmp.Compare[int]) })
	sc[0] = bestOf(5, ints, func(s []int) counts { return scanCopy(s, cmp.Compare[int]) })
	bin[0] = bestOf(5, ints, func(s []int) counts { return binary(s, cmp.Compare[int], true) })
	lin[1] = bestOf(5, strs, func(s []string) counts { return linear(s, strings.Compare) })
	sc[1] = bestOf(5, strs, func(s []string) counts { return scanCopy(s, strings.Compare) })
	bin[1] = bestOf(5, strs, func(s []string) counts { return binary(s, strings.Compare, true) })
	for t, name := range []string{"ints", "strings"} {
		fmt.Printf("  %-8s %10.2f %10.2f %10.2f   speed-up: block move %.1fx, then binary search %.1fx\n",
			name, ms(lin[t]), ms(sc[t]), ms(bin[t]), float64(lin[t])/float64(sc[t]), float64(sc[t])/float64(bin[t]))
	}
	fmt.Printf("  scan+copy (%d comparisons, block moves) takes %.1fx longer on strings than on ints\n",
		cl.comps, float64(sc[1])/float64(sc[0]))

	// (c) Stability: 16 records with keys in [0, 4); tag = input position.
	recs := make([]rec, 16)
	for i := range recs {
		recs[i] = rec{r.IntN(4), i}
	}
	fmt.Println("\n(c) stability, records key.tag (tag = input position)")
	fmt.Println("  input:      ", show(recs))
	for _, upper := range []bool{false, true} {
		s := slices.Clone(recs)
		binary(s, cmpKey, upper)
		name := "lower bound"
		if upper {
			name = "upper bound"
		}
		stable := firstViolation(s) == "none"
		fmt.Printf("  %s: %s\n", name, show(s))
		fmt.Printf("    sorted: %v  stable: %v  first violation: %s\n",
			slices.IsSortedFunc(s, cmpKey), stable, firstViolation(s))
	}
}
```

**Sample output:**

```
(a) n = 5000 random permutation, I = 6226935
  linear:  6231924 comparisons,  6226935 shifts
  binary:    54465 comparisons,  6226935 shifts
  shifts == I for both: true; binary makes 114x fewer comparisons; both sorted: true

(b) the same permutation as 46-byte strings with a 40-byte common prefix
  string counts equal int counts (same order): true
  scan+copy (linear search, one block copy) counts equal linear's: true
  timing in ms, best of 5, copy of the input not timed:
               linear  scan+copy     binary
  ints          11.62       6.83       0.68   speed-up: block move 1.7x, then binary search 10.1x
  strings       19.77      18.23       1.24   speed-up: block move 1.1x, then binary search 14.7x
  scan+copy (6231924 comparisons, block moves) takes 2.7x longer on strings than on ints

(c) stability, records key.tag (tag = input position)
  input:       3.0 1.1 0.2 1.3 0.4 2.5 0.6 3.7 3.8 1.9 3.10 3.11 0.12 0.13 1.14 1.15
  lower bound: 0.13 0.12 0.6 0.4 0.2 1.15 1.14 1.9 1.3 1.1 2.5 3.11 3.10 3.8 3.7 3.0
    sorted: true  stable: false  first violation: at 0: 0.13 before 0.12
  upper bound: 0.2 0.4 0.6 0.12 0.13 1.1 1.3 1.9 1.14 1.15 2.5 3.0 3.7 3.8 3.10 3.11
    sorted: true  stable: true  first violation: none
```

**Rust:**

```rust
use std::cmp::Ordering::{self, Greater, Less};
use std::time::{Duration, Instant};

// Binary insertion sort: the prefix s[..i] is sorted, so find the slot for
// s[i] by binary search -- about log2(i) comparisons instead of up to i --
// then move the elements in between one place right. The search makes the
// comparisons Theta(n log n), but the move is the same distance as before:
// shifts are still exactly I, the inversion count, Theta(n^2) on average.
//
// All sorts here take T: Copy, so `let x = s[i]` copies and `copy_within`
// can move a block (Go's copy). i64, &str (a borrowed string: pointer +
// length, like a Go string header) and the small record are all Copy.

#[derive(Clone, Copy, Default, PartialEq)]
struct Counts {
    comps: u64,
    shifts: u64,
}

// linear is the ordinary shift version.
fn linear<T: Copy>(s: &mut [T], cmp: impl Fn(&T, &T) -> Ordering) -> Counts {
    let mut c = Counts::default();
    for i in 1..s.len() {
        let (x, mut j) = (s[i], i);
        while j > 0 {
            c.comps += 1;
            if cmp(&x, &s[j - 1]) != Less {
                break; // stop at the first element <= x: stable
            }
            s[j] = s[j - 1];
            c.shifts += 1;
            j -= 1;
        }
        s[j] = x;
    }
    c
}

// scan_copy is a control for the timing table: the SAME linear search (same
// comparisons as linear), but the elements are moved afterwards by one
// copy_within instead of one at a time. It separates "fewer comparisons"
// from "block move".
fn scan_copy<T: Copy>(s: &mut [T], cmp: impl Fn(&T, &T) -> Ordering) -> Counts {
    let mut c = Counts::default();
    for i in 1..s.len() {
        let (x, mut j) = (s[i], i);
        while j > 0 {
            c.comps += 1;
            if cmp(&x, &s[j - 1]) != Less {
                break;
            }
            j -= 1;
        }
        s.copy_within(j..i, j + 1);
        s[j] = x;
        c.shifts += (i - j) as u64;
    }
    c
}

/// Go's sort.Search, line for line: the smallest j in [0, n) with pred(j)
/// true, or n. Ported rather than replaced by partition_point so that the
/// comparison counts match the Go program exactly (std bisects differently).
fn search(n: usize, mut pred: impl FnMut(usize) -> bool) -> usize {
    let (mut i, mut j) = (0, n);
    while i < j {
        let h = (i + j) / 2;
        if !pred(h) {
            i = h + 1;
        } else {
            j = h;
        }
    }
    i
}

#[derive(Clone, Copy, PartialEq)]
enum Bound {
    Lower,    // first slot whose element is >= x: BEFORE equal keys, not stable
    Upper,    // first slot whose element is > x (Go's sort.Search): stable
    StdUpper, // the same slot via std's partition_point(|y| y <= x)
}

fn binary<T: Copy>(s: &mut [T], cmp: impl Fn(&T, &T) -> Ordering, bound: Bound) -> Counts {
    let (mut comps, mut shifts) = (0, 0);
    let mut count = |a: &T, b: &T| {
        comps += 1;
        cmp(a, b)
    };
    for i in 1..s.len() {
        let x = s[i];
        let pos = match bound {
            // binary_search_by would return ANY equal element's index (its doc
            // says so) -- worse than Go's earliest; partition_point is exact.
            Bound::Lower => s[..i].partition_point(|y| count(y, &x) == Less),
            Bound::Upper => search(i, |j| count(&s[j], &x) == Greater),
            Bound::StdUpper => s[..i].partition_point(|y| count(y, &x) != Greater),
        };
        s.copy_within(pos..i, pos + 1); // one block move of i-pos elements
        s[pos] = x;
        shifts += (i - pos) as u64;
    }
    Counts { comps, shifts }
}

fn inversions(a: &[i64]) -> u64 {
    a.iter()
        .enumerate()
        .map(|(i, x)| a[i + 1..].iter().filter(|y| x > y).count() as u64)
        .sum()
}

// best_of times sort on a fresh copy of input (copy not timed), fastest of reps.
fn best_of<T: Copy>(reps: usize, input: &[T], sort: impl Fn(&mut [T]) -> Counts) -> Duration {
    let mut buf = input.to_vec();
    let mut best = Duration::MAX;
    for _ in 0..reps {
        buf.copy_from_slice(input);
        let t = Instant::now();
        std::hint::black_box(sort(std::hint::black_box(&mut buf)));
        best = best.min(t.elapsed());
    }
    best
}

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so every
// input below is exactly the one the Go program sorts.
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

#[derive(Clone, Copy)]
struct Rec {
    key: i64,
    tag: usize,
}

fn cmp_key(a: &Rec, b: &Rec) -> Ordering {
    a.key.cmp(&b.key)
}

fn show(rs: &[Rec]) -> String {
    let parts: Vec<String> = rs.iter().map(|r| format!("{}.{}", r.key, r.tag)).collect();
    parts.join(" ")
}

// first_violation: the first adjacent pair with equal keys in the wrong tag order.
fn first_violation(rs: &[Rec]) -> Option<String> {
    rs.windows(2)
        .position(|w| w[0].key == w[1].key && w[0].tag > w[1].tag)
        .map(|i| {
            let (a, b) = (rs[i], rs[i + 1]);
            format!("at {i}: {}.{} before {}.{}", a.key, a.tag, b.key, b.tag)
        })
}

fn main() {
    const N: usize = 5000;
    let mut r = Pcg::new(1, 2);
    let mut ints: Vec<i64> = (0..N as i64).collect();
    for i in (1..N).rev() {
        let j = r.int_n(i + 1);
        ints.swap(i, j);
    }
    let inv = inversions(&ints);
    let icmp = |a: &i64, b: &i64| a.cmp(b);

    println!("(a) n = {N} random permutation, I = {inv}");
    let (mut a, mut b) = (ints.clone(), ints.clone());
    let (cl, cb) = (linear(&mut a, icmp), binary(&mut b, icmp, Bound::Upper));
    println!(
        "  linear: {:>8} comparisons, {:>8} shifts",
        cl.comps, cl.shifts
    );
    println!(
        "  binary: {:>8} comparisons, {:>8} shifts",
        cb.comps, cb.shifts
    );
    println!(
        "  shifts == I for both: {}; binary makes {:.0}x fewer comparisons; both sorted: {}",
        cl.shifts == inv && cb.shifts == inv,
        cl.comps as f64 / cb.comps as f64,
        a.is_sorted() && b.is_sorted()
    );
    let mut p = ints.clone();
    let cp = binary(&mut p, icmp, Bound::StdUpper);
    println!(
        "  (Rust only) partition_point(|y| y <= x) instead: {} comparisons, {} shifts, same result: {}",
        cp.comps,
        cp.shifts,
        p == b
    );

    // The same permutation as strings that share a 40-byte prefix: the same
    // order, so the same counts, but every comparison now walks 40+ bytes.
    // `owned` holds the Strings; the sorts move &str borrows of them.
    let owned: Vec<String> = ints
        .iter()
        .map(|v| format!("customer-account-region-eu-west-0000000-{v:06}"))
        .collect();
    let strs: Vec<&str> = owned.iter().map(String::as_str).collect();
    let scmp = |a: &&str, b: &&str| a.cmp(b);
    let cs = linear(&mut strs.clone(), scmp);
    println!(
        "\n(b) the same permutation as {}-byte strings with a {}-byte common prefix",
        strs[0].len(),
        "customer-account-region-eu-west-0000000-".len()
    );
    println!(
        "  string counts equal int counts (same order): {}",
        cs == cl
    );
    println!(
        "  scan+copy (linear search, one block copy) counts equal linear's: {}",
        scan_copy(&mut strs.clone(), scmp) == cl
    );

    println!("  timing in ms, best of 5, copy of the input not timed:");
    println!(
        "  {:<8} {:>10} {:>10} {:>10}",
        "", "linear", "scan+copy", "binary"
    );
    let ms = |d: Duration| d.as_nanos() as f64 / 1e6;
    let lin = [
        best_of(5, &ints, |s| linear(s, icmp)),
        best_of(5, &strs, |s| linear(s, scmp)),
    ];
    let sc = [
        best_of(5, &ints, |s| scan_copy(s, icmp)),
        best_of(5, &strs, |s| scan_copy(s, scmp)),
    ];
    let bin = [
        best_of(5, &ints, |s| binary(s, icmp, Bound::Upper)),
        best_of(5, &strs, |s| binary(s, scmp, Bound::Upper)),
    ];
    for (t, name) in ["ints", "strings"].iter().enumerate() {
        println!(
            "  {:<8} {:>10.2} {:>10.2} {:>10.2}   speed-up: block move {:.1}x, then binary search {:.1}x",
            name,
            ms(lin[t]),
            ms(sc[t]),
            ms(bin[t]),
            lin[t].as_secs_f64() / sc[t].as_secs_f64(),
            sc[t].as_secs_f64() / bin[t].as_secs_f64()
        );
    }
    println!(
        "  scan+copy ({} comparisons, block moves) takes {:.1}x longer on strings than on ints",
        cl.comps,
        sc[1].as_secs_f64() / sc[0].as_secs_f64()
    );

    // (c) Stability: 16 records with keys in [0, 4); tag = input position.
    let recs: Vec<Rec> = (0..16)
        .map(|i| Rec {
            key: r.int_n(4) as i64,
            tag: i,
        })
        .collect();
    println!("\n(c) stability, records key.tag (tag = input position)");
    println!("  input:       {}", show(&recs));
    for (bound, name) in [(Bound::Lower, "lower bound"), (Bound::Upper, "upper bound")] {
        let mut s = recs.clone();
        binary(&mut s, cmp_key, bound);
        let v = first_violation(&s);
        println!("  {name}: {}", show(&s));
        println!(
            "    sorted: {}  stable: {}  first violation: {}",
            s.is_sorted_by(|a, b| cmp_key(a, b) != Greater),
            v.is_none(),
            v.as_deref().unwrap_or("none")
        );
    }
}
```

**Rust sample output:**

```
(a) n = 5000 random permutation, I = 6226935
  linear:  6231924 comparisons,  6226935 shifts
  binary:    54465 comparisons,  6226935 shifts
  shifts == I for both: true; binary makes 114x fewer comparisons; both sorted: true
  (Rust only) partition_point(|y| y <= x) instead: 61795 comparisons, 6226935 shifts, same result: true

(b) the same permutation as 46-byte strings with a 40-byte common prefix
  string counts equal int counts (same order): true
  scan+copy (linear search, one block copy) counts equal linear's: true
  timing in ms, best of 5, copy of the input not timed:
               linear  scan+copy     binary
  ints           1.65       2.14       0.60   speed-up: block move 0.8x, then binary search 3.6x
  strings       12.40      13.15       1.55   speed-up: block move 0.9x, then binary search 8.5x
  scan+copy (6231924 comparisons, block moves) takes 6.1x longer on strings than on ints

(c) stability, records key.tag (tag = input position)
  input:       3.0 1.1 0.2 1.3 0.4 2.5 0.6 3.7 3.8 1.9 3.10 3.11 0.12 0.13 1.14 1.15
  lower bound: 0.13 0.12 0.6 0.4 0.2 1.15 1.14 1.9 1.3 1.1 2.5 3.11 3.10 3.8 3.7 3.0
    sorted: true  stable: false  first violation: at 0: 0.13 before 0.12
  upper bound: 0.2 0.4 0.6 0.12 0.13 1.1 1.3 1.9 1.14 1.15 2.5 3.0 3.7 3.8 3.10 3.11
    sorted: true  stable: true  first violation: none
```

> **Go vs Rust:** All counts and all of section (c) match Go exactly. The upper bound is a line-for-line port of `sort.Search`. The extra `(Rust only)` line shows that std's `partition_point` finds the same slots with 61,795 comparisons instead of 54,465. It is built on `binary_search_by`, which halves while `size > 1` and then makes one more comparison, so it never stops early (`core/src/slice/mod.rs`). `binary_search_by` itself returns any equal element, so it cannot serve as the stable bound. Strings are sorted as `&str`, which is `Copy` like a Go string header, so `copy_within` moves them as a block.
> The timings tell a different story from Go's. With the comparator inlined, linear insertion on ints takes about 1.6 ms against Go's 11–12 ms. Binary insertion is then only about 2.7× faster on ints but about 8× faster on strings, which is the "worth it when comparisons are expensive" pattern Go did not show. The block move alone is slightly slower than shifting one at a time in Rust (0.8–0.9×). Plausible mechanism, not measured: the separate scan and copy walk the same memory twice.

**Complexity:** comparisons Θ(n log n), 54,465 against 6,231,924 at n = 5000 · moves are still exactly I = 6,226,935, which is Θ(n + I) and Θ(n²) on average · stable only with the upper bound: the lower bound reverses every group of equal keys · measured in Go, binary insertion is about 17× faster than linear insertion on ints and about 16× on strings. The scan+copy control shows where that comes from: dropping the comparisons gives about 10× on ints and about 14× on strings, and the block move alone gives about 1.7× on ints and only 1.0–1.1× on strings. A comparison made through a `cmp` function value is not cheap even for ints, so in Go the search pays off on both kinds of key.

---

## 10. Multi-key sorting needs stability

`🟡 medium` · *Stability as correctness*

To sort a table by department and, inside each department, by name, sort by the minor key first and then
**stably** by the major key: the stable pass leaves equal departments in the order the first pass made, which was
alphabetical. Replace that second pass with an unstable sort and the first pass is silently destroyed — the
output still looks grouped, but the names inside each group are scrambled. The fix that needs no stability is one
**total** comparator (`cmp.Or(dept, name)`), which makes any sort's output unique. Cost: two Θ(n log n) passes
for the stable route, one for the composite comparator.

**Steps:**

1. Generate 200 employees with distinct names, a PCG-chosen department and a score in [0, 10).
2. Sort them three ways — two stable passes, stable-then-unstable, one `cmp.Or` comparator — and check whether the
   names are sorted inside every department.
3. Sort by score alone (heavy ties) unstably, with an ID tie-breaker, and stably, and see which two agree.

```go
package main

import (
	"cmp"
	"fmt"
	"math/rand/v2"
	"slices"
	"strings"
)

// Multi-key sorting by successive passes: sort by the MINOR key first, then
// STABLY by the major key. The stable pass keeps equal departments in the
// order the first pass left them -- alphabetical -- so you get departments
// grouped with names sorted inside each. That is the spreadsheet trick of
// clicking "Name" and then "Dept". With an unstable second pass, the tie
// order inside a department is whatever the algorithm happens to leave.

type Employee struct {
	ID    int // position in the input: 0..n-1
	Name  string
	Dept  string
	Score int
}

func byName(a, b Employee) int  { return strings.Compare(a.Name, b.Name) }
func byDept(a, b Employee) int  { return strings.Compare(a.Dept, b.Dept) }
func byScore(a, b Employee) int { return cmp.Compare(a.Score, b.Score) }

// staff builds n = 20 x 10 = 200 distinct names in a shuffled order and gives
// each a random department and a score in [0, 10) -- so ties are everywhere.
func staff() []Employee {
	first := []string{"Ada", "Ben", "Cleo", "Dan", "Eva", "Finn", "Gus", "Hana", "Ivo", "Jun",
		"Kai", "Lena", "Max", "Nia", "Omar", "Pia", "Quin", "Rosa", "Sam", "Tess"}
	last := []string{"Abbott", "Brook", "Cole", "Diaz", "Ernst", "Frey", "Grant", "Hale", "Ito", "Jones"}
	depts := []string{"eng", "hr", "legal", "ops", "sales"}
	r := rand.New(rand.NewPCG(12, 10))
	names := make([]string, 0, len(first)*len(last))
	for _, l := range last {
		for _, f := range first {
			names = append(names, f+" "+l)
		}
	}
	for i := len(names) - 1; i > 0; i-- { // Fisher-Yates on r.IntN(i+1)
		j := r.IntN(i + 1)
		names[i], names[j] = names[j], names[i]
	}
	es := make([]Employee, len(names))
	for i, nm := range names {
		es[i] = Employee{ID: i, Name: nm, Dept: depts[r.IntN(len(depts))], Score: r.IntN(10)}
	}
	return es
}

// twoPass: sort by name (stable or not -- names are distinct, so it cannot
// matter), then by department with the given second-pass sort.
func twoPass(es []Employee, second func([]Employee, func(a, b Employee) int)) []Employee {
	out := slices.Clone(es)
	slices.SortStableFunc(out, byName)
	second(out, byDept)
	return out
}

// violations counts adjacent pairs in the same department whose names are out
// of order, and returns the index of the first one (-1 if none) and the number
// of adjacent same-department pairs there are to check.
func violations(es []Employee) (count, first, pairs int) {
	first = -1
	for i := 0; i+1 < len(es); i++ {
		if es[i].Dept != es[i+1].Dept {
			continue
		}
		pairs++
		if es[i].Name > es[i+1].Name {
			if first < 0 {
				first = i
			}
			count++
		}
	}
	return count, first, pairs
}

// idsWithScore lists the IDs of the employees with score s, in output order.
func idsWithScore(es []Employee, s, max int) []int {
	var ids []int
	for _, e := range es {
		if e.Score == s && len(ids) < max {
			ids = append(ids, e.ID)
		}
	}
	return ids
}

func main() {
	es := staff()
	n := len(es)
	stable := slices.SortStableFunc[[]Employee, Employee]
	unstable := slices.SortFunc[[]Employee, Employee]
	fmt.Printf("%d employees, 5 departments, scores in [0, 10)\n\n", n)

	// (a) Two stable passes: name, then department.
	a := twoPass(es, stable)
	fmt.Println("(a) SortStableFunc by name, then SortStableFunc by dept -- first 8 rows:")
	for _, e := range a[:8] {
		fmt.Printf("    %-6s %-12s score %d\n", e.Dept, e.Name, e.Score)
	}
	va, _, _ := violations(a)
	fmt.Printf("    depts grouped? %v   names sorted within each dept? %v\n\n",
		slices.IsSortedFunc(a, byDept), va == 0)

	// (b) Same first pass, but the second pass is the unstable SortFunc.
	b := twoPass(es, unstable)
	vb, fb, pb := violations(b)
	fmt.Println("(b) SortStableFunc by name, then SortFunc by dept:")
	fmt.Printf("    depts grouped? %v   names sorted within each dept? %v\n",
		slices.IsSortedFunc(b, byDept), vb == 0)
	if fb >= 0 {
		fmt.Printf("    first violation at rows %d-%d: %s %q > %q\n",
			fb, fb+1, b[fb].Dept, b[fb].Name, b[fb+1].Name)
	}
	fmt.Printf("    adjacent same-dept pairs out of order: %d of %d\n", vb, pb)
	// The smallest prefix es[:k] on which the unstable second pass goes wrong.
	// This number belongs to the library, not to sorting: it is where pdqsort
	// stops being plain insertion sort (which is stable by accident).
	for k := 2; k <= n; k++ {
		if v, _, _ := violations(twoPass(es[:k], unstable)); v > 0 {
			fmt.Printf("    smallest prefix of the input where (b) goes wrong: n = %d\n", k)
			break
		}
	}
	fmt.Println("    (which ties move is implementation-defined: another library gives other numbers)")
	fmt.Println()

	// (c) No stability needed: one total comparator, one unstable sort.
	c := slices.Clone(es)
	slices.SortFunc(c, func(x, y Employee) int {
		return cmp.Or(byDept(x, y), byName(x, y)) // first non-zero wins
	})
	fmt.Println("(c) one SortFunc with cmp.Or(dept, name):")
	fmt.Printf("    identical to (a)? %v\n\n", slices.Equal(c, a))

	// (d) Sorting by score alone: about 20 employees share each score.
	u := slices.Clone(es)
	slices.SortFunc(u, byScore)
	t := slices.Clone(es)
	slices.SortFunc(t, func(x, y Employee) int {
		return cmp.Or(byScore(x, y), cmp.Compare(x.ID, y.ID)) // ties by input order
	})
	s := slices.Clone(es)
	slices.SortStableFunc(s, byScore)
	differ := 0
	for i := range u {
		if u[i] != s[i] {
			differ++
		}
	}
	fmt.Println("(d) by score; IDs of the first 8 employees with score 0, in output order:")
	fmt.Printf("    SortFunc(score)            %v\n", idsWithScore(u, 0, 8))
	fmt.Printf("    SortFunc(score, then ID)   %v\n", idsWithScore(t, 0, 8))
	fmt.Printf("    SortStableFunc(score)      %v\n", idsWithScore(s, 0, 8))
	fmt.Printf("    tie-broken == stable? %v   unstable == stable? %v (%d of %d rows differ)\n",
		slices.Equal(t, s), slices.Equal(u, s), differ, n)
}
```

**Output:**

```
200 employees, 5 departments, scores in [0, 10)

(a) SortStableFunc by name, then SortStableFunc by dept -- first 8 rows:
    eng    Ada Diaz     score 4
    eng    Ada Frey     score 4
    eng    Ben Grant    score 0
    eng    Ben Ito      score 4
    eng    Cleo Cole    score 3
    eng    Cleo Ito     score 1
    eng    Cleo Jones   score 8
    eng    Dan Abbott   score 7
    depts grouped? true   names sorted within each dept? true

(b) SortStableFunc by name, then SortFunc by dept:
    depts grouped? true   names sorted within each dept? false
    first violation at rows 0-1: eng "Lena Ito" > "Finn Jones"
    adjacent same-dept pairs out of order: 101 of 195
    smallest prefix of the input where (b) goes wrong: n = 13
    (which ties move is implementation-defined: another library gives other numbers)

(c) one SortFunc with cmp.Or(dept, name):
    identical to (a)? true

(d) by score; IDs of the first 8 employees with score 0, in output order:
    SortFunc(score)            [18 170 41 175 161 115 6 160]
    SortFunc(score, then ID)   [6 14 16 18 19 41 83 103]
    SortStableFunc(score)      [6 14 16 18 19 41 83 103]
    tie-broken == stable? true   unstable == stable? false (188 of 200 rows differ)
```

**Rust:**

```rust
use std::cmp::Ordering;

// Multi-key sorting by successive passes: sort by the MINOR key first, then
// STABLY by the major key. The stable pass keeps equal departments in the
// order the first pass left them -- alphabetical -- so you get departments
// grouped with names sorted inside each. In Rust the stability is in the name:
// `sort_by` is stable (driftsort), `sort_unstable_by` is not (ipnsort).

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so this
// program sees exactly the 200 employees the Go program sees.
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

#[derive(Clone, PartialEq, Eq)]
struct Employee {
    id: usize, // position in the input: 0..n-1
    name: String,
    dept: &'static str,
    score: u32,
}

// Comparators borrow both records: no key is copied or cloned.
fn by_name(a: &Employee, b: &Employee) -> Ordering {
    a.name.cmp(&b.name)
}
fn by_dept(a: &Employee, b: &Employee) -> Ordering {
    a.dept.cmp(b.dept)
}
fn by_score(a: &Employee, b: &Employee) -> Ordering {
    a.score.cmp(&b.score)
}

// staff builds n = 20 x 10 = 200 distinct names in a shuffled order and gives
// each a random department and a score in [0, 10) -- so ties are everywhere.
fn staff() -> Vec<Employee> {
    let first = [
        "Ada", "Ben", "Cleo", "Dan", "Eva", "Finn", "Gus", "Hana", "Ivo", "Jun", "Kai", "Lena",
        "Max", "Nia", "Omar", "Pia", "Quin", "Rosa", "Sam", "Tess",
    ];
    let last = [
        "Abbott", "Brook", "Cole", "Diaz", "Ernst", "Frey", "Grant", "Hale", "Ito", "Jones",
    ];
    let depts = ["eng", "hr", "legal", "ops", "sales"];
    let mut r = Pcg::new(12, 10);
    let mut names: Vec<String> = last
        .iter()
        .flat_map(|l| first.iter().map(move |f| format!("{f} {l}")))
        .collect();
    for i in (1..names.len()).rev() {
        // Fisher-Yates on int_n(i+1), exactly as the Go program does it
        let j = r.int_n(i as u64 + 1) as usize;
        names.swap(i, j);
    }
    names
        .into_iter()
        .enumerate()
        .map(|(id, name)| Employee {
            id,
            name,
            dept: depts[r.int_n(depts.len() as u64) as usize],
            score: r.int_n(10) as u32,
        })
        .collect()
}

// two_pass: sort by name (names are distinct, so stability cannot matter
// here), then by department with the given second-pass sort.
fn two_pass(es: &[Employee], second: impl Fn(&mut [Employee])) -> Vec<Employee> {
    let mut out = es.to_vec();
    out.sort_by(by_name);
    second(&mut out);
    out
}

struct Violations {
    count: usize,         // adjacent same-dept pairs whose names are out of order
    first: Option<usize>, // index of the first such pair
    pairs: usize,         // adjacent same-dept pairs there are to check
}

fn violations(es: &[Employee]) -> Violations {
    let mut v = Violations {
        count: 0,
        first: None,
        pairs: 0,
    };
    for (i, w) in es.windows(2).enumerate() {
        if w[0].dept != w[1].dept {
            continue;
        }
        v.pairs += 1;
        if w[0].name > w[1].name {
            v.first.get_or_insert(i);
            v.count += 1;
        }
    }
    v
}

// ids_with_score lists the IDs of the employees with score s, in output order.
fn ids_with_score(es: &[Employee], s: u32, max: usize) -> String {
    let ids: Vec<String> = es
        .iter()
        .filter(|e| e.score == s)
        .take(max)
        .map(|e| e.id.to_string())
        .collect();
    format!("[{}]", ids.join(" ")) // Go's %v spelling, not Rust's {:?}
}

fn main() {
    let es = staff();
    let n = es.len();
    let stable = |v: &mut [Employee]| v.sort_by(by_dept);
    let unstable = |v: &mut [Employee]| v.sort_unstable_by(by_dept);
    println!("{n} employees, 5 departments, scores in [0, 10)\n");

    // (a) Two stable passes: name, then department.
    let a = two_pass(&es, stable);
    println!("(a) sort_by name, then sort_by dept -- first 8 rows:");
    for e in &a[..8] {
        println!("    {:<6} {:<12} score {}", e.dept, e.name, e.score);
    }
    println!(
        "    depts grouped? {}   names sorted within each dept? {}\n",
        a.is_sorted_by(|x, y| by_dept(x, y).is_le()),
        violations(&a).count == 0
    );

    // (b) Same first pass, but the second pass is the unstable sort.
    let b = two_pass(&es, unstable);
    let vb = violations(&b);
    println!("(b) sort_by name, then sort_unstable_by dept:");
    println!(
        "    depts grouped? {}   names sorted within each dept? {}",
        b.is_sorted_by(|x, y| by_dept(x, y).is_le()),
        vb.count == 0
    );
    if let Some(f) = vb.first {
        println!(
            "    first violation at rows {}-{}: {} {:?} > {:?}",
            f,
            f + 1,
            b[f].dept,
            b[f].name,
            b[f + 1].name
        );
    }
    println!(
        "    adjacent same-dept pairs out of order: {} of {}",
        vb.count, vb.pairs
    );
    // The smallest prefix es[..k] on which the unstable second pass goes wrong.
    // This number belongs to the library, not to sorting: sort_unstable_by is
    // plain insertion sort (stable by accident) for len <= 20, and above that
    // its quicksort hands every range of <= 32 elements of this type (56 bytes,
    // no interior mutability) to small_sort_general, which is built from
    // stable pieces (sort4_stable, sort8_stable, insertion, a merge). So ties
    // can only move once a partition step runs, on more than 32 elements
    // (core/src/slice/sort/shared/smallsort.rs, read in the source).
    if let Some(k) = (2..=n).find(|&k| violations(&two_pass(&es[..k], unstable)).count > 0) {
        println!("    smallest prefix of the input where (b) goes wrong: n = {k}");
    }
    println!(
        "    (which ties move is implementation-defined: another library gives other numbers)"
    );
    println!();

    // (c) No stability needed: one total comparator, one unstable sort.
    // `then_with` evaluates the name comparison only when the depts tie --
    // the same short-circuit as Go's cmp.Or. (sort_by_key would need an owned
    // key, `|e| (e.dept, e.name.clone())`, i.e. one String clone per call:
    // a key function cannot return a borrow of its argument.)
    let mut c = es.clone();
    c.sort_unstable_by(|x, y| by_dept(x, y).then_with(|| by_name(x, y)));
    println!("(c) one sort_unstable_by with dept.then_with(name):");
    println!("    identical to (a)? {}\n", c == a);

    // (d) Sorting by score alone: about 20 employees share each score.
    let mut u = es.clone();
    u.sort_unstable_by(by_score);
    let mut t = es.clone();
    t.sort_unstable_by(|x, y| by_score(x, y).then(x.id.cmp(&y.id))); // ties by input order
    let mut s = es.clone();
    s.sort_by(by_score);
    let differ = u.iter().zip(&s).filter(|(x, y)| x != y).count();
    println!("(d) by score; IDs of the first 8 employees with score 0, in output order:");
    println!(
        "    sort_unstable_by(score)        {}",
        ids_with_score(&u, 0, 8)
    );
    println!(
        "    sort_unstable_by(score, id)    {}",
        ids_with_score(&t, 0, 8)
    );
    println!(
        "    sort_by(score)                 {}",
        ids_with_score(&s, 0, 8)
    );
    println!(
        "    tie-broken == stable? {}   unstable == stable? {} ({} of {} rows differ)",
        t == s,
        u == s,
        differ,
        n
    );
}
```

**Rust output:**

```
200 employees, 5 departments, scores in [0, 10)

(a) sort_by name, then sort_by dept -- first 8 rows:
    eng    Ada Diaz     score 4
    eng    Ada Frey     score 4
    eng    Ben Grant    score 0
    eng    Ben Ito      score 4
    eng    Cleo Cole    score 3
    eng    Cleo Ito     score 1
    eng    Cleo Jones   score 8
    eng    Dan Abbott   score 7
    depts grouped? true   names sorted within each dept? true

(b) sort_by name, then sort_unstable_by dept:
    depts grouped? true   names sorted within each dept? false
    first violation at rows 0-1: eng "Tess Jones" > "Ada Frey"
    adjacent same-dept pairs out of order: 66 of 195
    smallest prefix of the input where (b) goes wrong: n = 33
    (which ties move is implementation-defined: another library gives other numbers)

(c) one sort_unstable_by with dept.then_with(name):
    identical to (a)? true

(d) by score; IDs of the first 8 employees with score 0, in output order:
    sort_unstable_by(score)        [186 6 14 16 18 19 41 83]
    sort_unstable_by(score, id)    [6 14 16 18 19 41 83 103]
    sort_by(score)                 [6 14 16 18 19 41 83 103]
    tie-broken == stable? true   unstable == stable? false (187 of 200 rows differ)
```

> **Go vs Rust:** (a), (c) and the tie-broken/stable rows of (d) are character-identical to Go (same PCG stream, and a total order or a stable sort leaves exactly one correct answer). Only the unstable rows differ, because the unstable algorithms differ: Go's pdqsort first scrambles ties at n = 13 (insertion sort only up to 12) and leaves 101 of 195 pairs out of order; Rust's ipnsort first does so at n = 33 (insertion sort up to 20, then a small-sort built from stable pieces up to 32 for this 56-byte type — read in `smallsort.rs`) and leaves 66. `then_with` is Rust's `cmp.Or`; `sort_by_key` would need an owned (cloned) key, which is why the comparators borrow.

**Complexity:** two stable passes = two Θ(n log n) sorts; one composite comparator = one sort. Measured: the stable route and `cmp.Or` give identical output, while an unstable second pass leaves 101 of 195 same-department neighbours out of order and first goes wrong at n = 13 — just past pdqsort's insertion-sort cutoff of 12. With a total order (score, then ID), unstable `SortFunc` equals `SortStableFunc` row for row.

---

## 11. The comparator contract

`🟡 medium` · *Strict weak ordering*

A sort is only as correct as its comparator, which must be a **strict weak ordering**: irreflexive (`less(a, a)` is
false), transitive, and with "neither is less" transitive too. This example breaks it three real ways and checks
each output instead of trusting it: `<=` instead of `<` (still sorted, silently unstable — even under
`SortStableFunc`), Go's `return a - b` (wraps around near the int64 limits and returns the wrong sign), and `NaN`
under a naive `<`/`>` comparator (NaN is "equal" to everything, so incomparability stops being transitive). A broken
comparator does not change the cost — still O(n log n) or O(n²) — it makes the result **undefined**, which is worse
than reliably wrong: one input can come out right by luck, so the last section counts over 1000 random inputs.

**Steps:**

1. Sort ten tagged pairs with a correct and a `<=` comparator, through your insertion sort and `SortStableFunc`.
2. Sort 16 ints near the int64 limits with `a - b` and with `cmp.Compare`, and floats with three NaNs naively, with
   `slices.Sort` and with `cmp.Compare`; print `sorted?` for each.
3. Run the three broken comparators over 1000 random inputs per size, past the insertion-sort cutoffs, and count
   unsorted results and panics.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"strings"
)

// A sort is only as correct as its comparator. SortFunc's doc: "cmp ... must
// be a strict weak ordering": irreflexive (less(a, a) is false), transitive,
// and "neither is less" (incomparability) is transitive too. Three common ways
// to break it, each checked by looking at the output, not by trusting it.

type Pair struct {
	Key int
	Tag byte // input order: 'a', 'b', ...
}

// insertion is the lesson's shift-based insertion sort with a less function.
func insertion[E any](s []E, less func(a, b E) bool) {
	for i := 1; i < len(s); i++ {
		x, j := s[i], i
		for ; j > 0 && less(x, s[j-1]); j-- {
			s[j] = s[j-1]
		}
		s[j] = x
	}
}

// join formats a slice with f, space-separated (the Rust twin uses the same).
func join[E any](xs []E, f func(E) string) string {
	out := make([]string, len(xs))
	for i, x := range xs {
		out[i] = f(x)
	}
	return strings.Join(out, " ")
}

func pairStr(p Pair) string { return fmt.Sprintf("%d%c", p.Key, p.Tag) }

func sortedAndStable(ps []Pair) (sorted, stable bool) {
	sorted, stable = true, true
	for i := 1; i < len(ps); i++ {
		if ps[i-1].Key > ps[i].Key {
			sorted = false
		} else if ps[i-1].Key == ps[i].Key && ps[i-1].Tag > ps[i].Tag {
			stable = false
		}
	}
	return sorted, stable
}

// name prints the huge values by their distance from the int64 limits.
func name(v int) string {
	switch {
	case v >= math.MaxInt64-100:
		return fmt.Sprintf("MAX-%d", math.MaxInt64-v)
	case v <= math.MinInt64+100:
		return fmt.Sprintf("MIN+%d", v-math.MinInt64)
	}
	return fmt.Sprint(v)
}

func floatStr(v float64) string {
	if math.IsNaN(v) && math.Signbit(v) {
		return "-NaN" // fmt prints every NaN as "NaN"; show the sign bit
	}
	return fmt.Sprint(v)
}

// numbersInOrder: are the non-NaN values ascending (wherever the NaNs went)?
func numbersInOrder(xs []float64) bool {
	return slices.IsSorted(slices.DeleteFunc(slices.Clone(xs), math.IsNaN))
}

func subtract(a, b int) int { return a - b } // the classic bug: wraps around
func naive(a, b float64) int { // NaN compares "equal" to everything
	if a < b {
		return -1
	} else if a > b {
		return 1
	}
	return 0
}

func sortWith[E any](s []E, cmp func(a, b E) int, stable bool) {
	if stable {
		slices.SortStableFunc(s, cmp)
	} else {
		slices.SortFunc(s, cmp)
	}
}

// tryIt runs sort and reports whether it panicked (Go's sorts never did here).
func tryIt(sort func()) (panicked bool) {
	defer func() { panicked = recover() != nil }()
	sort()
	return false
}

func main() {
	// (1) "<=" instead of "<": less(a, a) is true, so equal keys look "less".
	in := []Pair{{3, 'a'}, {1, 'b'}, {2, 'c'}, {1, 'd'}, {3, 'e'}, {2, 'f'}, {1, 'g'}, {2, 'h'}, {3, 'i'}, {1, 'j'}}
	lt := func(a, b Pair) bool { return a.Key < b.Key }
	le := func(a, b Pair) bool { return a.Key <= b.Key }
	cmpLT := func(a, b Pair) int { return cmp.Compare(a.Key, b.Key) }
	cmpLE := func(a, b Pair) int { // "a <= b, so a goes first": never says 0
		if a.Key <= b.Key {
			return -1
		}
		return 1
	}
	fmt.Println("(1) comparator says \"less\" for equal keys; input:", join(in, pairStr))
	fmt.Printf("    less(1b, 1b) = %v -- irreflexivity is broken\n", le(in[1], in[1]))
	row1 := func(label string, sortIt func(s []Pair, count *int)) {
		s, calls := slices.Clone(in), 0
		sortIt(s, &calls)
		sorted, stable := sortedAndStable(s)
		fmt.Printf("    %-27s %s  sorted? %-5v stable? %-5v calls %d\n", label, join(s, pairStr), sorted, stable, calls)
	}
	row1("insertion, a < b", func(s []Pair, c *int) { insertion(s, func(a, b Pair) bool { *c++; return lt(a, b) }) })
	row1("insertion, a <= b", func(s []Pair, c *int) { insertion(s, func(a, b Pair) bool { *c++; return le(a, b) }) })
	row1("SortStableFunc, Compare", func(s []Pair, c *int) {
		slices.SortStableFunc(s, func(a, b Pair) int { *c++; return cmpLT(a, b) })
	})
	row1("SortStableFunc, <= style", func(s []Pair, c *int) {
		slices.SortStableFunc(s, func(a, b Pair) int { *c++; return cmpLE(a, b) })
	})
	fmt.Println()

	// (2) "return a - b" overflows: MAX - (-1) wraps to a negative number.
	ints := []int{5, math.MaxInt64, -3, math.MinInt64 + 1, 0, math.MaxInt64 - 2, 7, math.MinInt64,
		-8, math.MaxInt64 - 1, 2, math.MinInt64 + 2, 1, -1, 9, 4}
	fmt.Printf("(2) cmp = a - b on %d ints near the int64 limits\n", len(ints))
	fmt.Printf("    cmp(MAX, -1) = %d: a - b claims MAX < -1\n", subtract(math.MaxInt64, -1))
	row2 := func(label string, sortIt func([]int)) {
		s := slices.Clone(ints)
		sortIt(s)
		fmt.Printf("    %-22s sorted? %-5v %s\n", label, slices.IsSorted(s), join(s, name))
	}
	row2("insertion, a - b", func(s []int) { insertion(s, func(a, b int) bool { return subtract(a, b) < 0 }) })
	row2("SortFunc, a - b", func(s []int) { slices.SortFunc(s, subtract) })
	row2("SortFunc, cmp.Compare", func(s []int) { slices.SortFunc(s, cmp.Compare[int]) })
	// A broken cmp makes the result undefined, not reliably wrong: on one input
	// a sort can get lucky (here pdqsort, unlike insertion sort, happened to end
	// up sorted). That is what lets the bug survive tests -- (4) counts it.
	fmt.Println()

	// (3) NaN under a naive comparator: NaN is "equal" to 1 and to 9, but 1 is
	// not equal to 9 -- incomparability is not transitive.
	nan := math.NaN()
	fl := []float64{3, nan, 1, 4, math.Copysign(nan, -1), 1.5, 9, 2, nan, 6, 5, 0}
	fmt.Println("(3) floats with 3 NaNs (one with the sign bit set):", join(fl, floatStr))
	row3 := func(label string, sortIt func([]float64)) {
		s := slices.Clone(fl)
		sortIt(s)
		fmt.Printf("    %-22s numbers in order? %-5v %s\n", label, numbersInOrder(s), join(s, floatStr))
	}
	row3("SortFunc, naive < >", func(s []float64) { slices.SortFunc(s, naive) })
	row3("slices.Sort", slices.Sort[[]float64])
	row3("SortFunc, cmp.Compare", func(s []float64) { slices.SortFunc(s, cmp.Compare[float64]) })
	fmt.Println()

	// (4) The same three comparators on 1000 random inputs per size, past the
	// insertion-sort cutoffs (12 for SortFunc, blocks of 20 for SortStableFunc).
	// Each trial draws an input for the comparator under test and runs both
	// sorts on copies of it.
	r := rand.New(rand.NewPCG(12, 11))
	fmt.Println("(4) 1000 random inputs per n: unsorted results / panics")
	fmt.Printf("    %-6s %4s  %-13s %s\n", "cmp", "n", "SortFunc", "SortStableFunc")
	for _, c := range []string{"<=", "a - b", "naive"} {
		for _, n := range []int{20, 21, 32, 64, 200} {
			var bad, pan [2]int
			for range 1000 {
				ps, is, fs := make([]Pair, n), make([]int, n), make([]float64, n)
				for i := range n {
					switch c {
					case "<=":
						ps[i] = Pair{r.IntN(4), byte(i)}
					case "a - b": // a third each: near MIN, near MAX, small
						switch r.IntN(3) {
						case 0:
							is[i] = math.MinInt64 + r.IntN(50)
						case 1:
							is[i] = math.MaxInt64 - r.IntN(50)
						default:
							is[i] = r.IntN(100) - 50
						}
					case "naive": // one in 8 is NaN
						fs[i] = float64(r.IntN(1000))
						if r.IntN(8) == 0 {
							fs[i] = nan
						}
					}
				}
				for k, stable := range []bool{false, true} {
					var p, ok bool
					switch c {
					case "<=":
						s := slices.Clone(ps)
						p = tryIt(func() { sortWith(s, cmpLE, stable) })
						ok, _ = sortedAndStable(s)
					case "a - b":
						s := slices.Clone(is)
						p = tryIt(func() { sortWith(s, subtract, stable) })
						ok = slices.IsSorted(s)
					case "naive":
						s := slices.Clone(fs)
						p = tryIt(func() { sortWith(s, naive, stable) })
						ok = numbersInOrder(s)
					}
					if p {
						pan[k]++
					} else if !ok {
						bad[k]++
					}
				}
			}
			fmt.Printf("    %-6s %4d  %4d / %-6d %4d / %d\n", c, n, bad[0], pan[0], bad[1], pan[1])
		}
	}
}
```

**Output:**

```
(1) comparator says "less" for equal keys; input: 3a 1b 2c 1d 3e 2f 1g 2h 3i 1j
    less(1b, 1b) = true -- irreflexivity is broken
    insertion, a < b            1b 1d 1g 1j 2c 2f 2h 3a 3e 3i  sorted? true  stable? true  calls 26
    insertion, a <= b           1j 1g 1d 1b 2h 2f 2c 3i 3e 3a  sorted? true  stable? false calls 35
    SortStableFunc, Compare     1b 1d 1g 1j 2c 2f 2h 3a 3e 3i  sorted? true  stable? true  calls 26
    SortStableFunc, <= style    1j 1g 1d 1b 2h 2f 2c 3i 3e 3a  sorted? true  stable? false calls 35

(2) cmp = a - b on 16 ints near the int64 limits
    cmp(MAX, -1) = -9223372036854775808: a - b claims MAX < -1
    insertion, a - b       sorted? false 5 MAX-0 MIN+1 -3 0 7 MAX-2 MAX-1 MIN+0 MIN+2 -8 -1 1 2 4 9
    SortFunc, a - b        sorted? true  MIN+0 MIN+1 MIN+2 -8 -3 -1 0 1 2 4 5 7 9 MAX-2 MAX-1 MAX-0
    SortFunc, cmp.Compare  sorted? true  MIN+0 MIN+1 MIN+2 -8 -3 -1 0 1 2 4 5 7 9 MAX-2 MAX-1 MAX-0

(3) floats with 3 NaNs (one with the sign bit set): 3 NaN 1 4 -NaN 1.5 9 2 NaN 6 5 0
    SortFunc, naive < >    numbers in order? false 3 NaN 1 4 -NaN 1.5 2 9 NaN 0 5 6
    slices.Sort            numbers in order? true  NaN -NaN NaN 0 1 1.5 2 3 4 5 6 9
    SortFunc, cmp.Compare  numbers in order? true  NaN -NaN NaN 0 1 1.5 2 3 4 5 6 9

(4) 1000 random inputs per n: unsorted results / panics
    cmp       n  SortFunc      SortStableFunc
    <=       20     0 / 0         0 / 0
    <=       21     0 / 0         0 / 0
    <=       32     0 / 0         0 / 0
    <=       64     0 / 0         0 / 0
    <=      200     0 / 0         0 / 0
    a - b    20   951 / 0      1000 / 0
    a - b    21   970 / 0      1000 / 0
    a - b    32   975 / 0      1000 / 0
    a - b    64   992 / 0      1000 / 0
    a - b   200   995 / 0      1000 / 0
    naive    20   730 / 0       900 / 0
    naive    21   716 / 0       903 / 0
    naive    32   791 / 0       972 / 0
    naive    64   850 / 0       998 / 0
    naive   200   923 / 0      1000 / 0
```

**Rust:**

```rust
use std::cmp::Ordering::{self, Equal, Greater, Less};
use std::panic;

// A sort is only as correct as its comparator. Rust's docs ask for a TOTAL
// order (Ord); what sorting really needs is a strict weak ordering:
// irreflexive (less(a, a) is false), transitive, and "neither is less"
// (incomparability) transitive too. And Rust's sorts document a consequence Go's
// do not: they "may panic if `compare` does not implement a total order".

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so section (4)
// sees exactly the 15,000 random inputs the Go program sees.
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

#[derive(Clone, Copy)]
struct Pair {
    key: u64,
    tag: u8, // input order: b'a', b'b', ...
}

// insertion is the lesson's shift-based insertion sort with a less function.
fn insertion<T: Copy>(s: &mut [T], mut less: impl FnMut(&T, &T) -> bool) {
    for i in 1..s.len() {
        let (x, mut j) = (s[i], i);
        while j > 0 && less(&x, &s[j - 1]) {
            s[j] = s[j - 1];
            j -= 1;
        }
        s[j] = x;
    }
}

// join formats a slice with f, space-separated (the Go program uses the same).
fn join<T>(xs: &[T], f: impl Fn(&T) -> String) -> String {
    xs.iter().map(f).collect::<Vec<_>>().join(" ")
}

fn pair_str(p: &Pair) -> String {
    format!("{}{}", p.key, p.tag as char)
}

fn sorted_and_stable(ps: &[Pair]) -> (bool, bool) {
    let (mut sorted, mut stable) = (true, true);
    for w in ps.windows(2) {
        if w[0].key > w[1].key {
            sorted = false;
        } else if w[0].key == w[1].key && w[0].tag > w[1].tag {
            stable = false;
        }
    }
    (sorted, stable)
}

// name prints the huge values by their distance from the i64 limits.
fn name(&v: &i64) -> String {
    if v >= i64::MAX - 100 {
        format!("MAX-{}", i64::MAX - v)
    } else if v <= i64::MIN + 100 {
        format!("MIN+{}", v - i64::MIN)
    } else {
        v.to_string()
    }
}

fn float_str(v: &f64) -> String {
    if v.is_nan() && v.is_sign_negative() {
        "-NaN".to_string() // `{}` prints every NaN as "NaN"; show the sign bit
    } else {
        v.to_string()
    }
}

// numbers_in_order: are the non-NaN values ascending (wherever the NaNs went)?
fn numbers_in_order(xs: &[f64]) -> bool {
    xs.iter().filter(|v| !v.is_nan()).is_sorted()
}

// Go's `return a - b`. Plain `a - b` would panic on overflow in a debug build
// and wrap in a release build; wrapping_sub states the wrap the bug relies on.
// The idiomatic `a.cmp(b)` cannot overflow at all.
fn subtract(a: &i64, b: &i64) -> Ordering {
    a.wrapping_sub(*b).cmp(&0)
}

// Go's naive comparator: NaN compares "equal" to everything.
fn naive(a: &f64, b: &f64) -> Ordering {
    a.partial_cmp(b).unwrap_or(Equal)
}

fn sort_with<T>(s: &mut [T], cmp: impl FnMut(&T, &T) -> Ordering, stable: bool) {
    if stable {
        s.sort_by(cmp);
    } else {
        s.sort_unstable_by(cmp);
    }
}

// try_it runs f and catches a panic, returning its message. A panic unwinds (it
// is not an abort), so catch_unwind can stop it; the default hook, which would
// print "thread 'main' panicked at ..." to stderr, is muted meanwhile. Rust
// guarantees the slice still holds all its original elements afterwards.
fn try_it(f: impl FnOnce()) -> Option<String> {
    let hook = panic::take_hook();
    panic::set_hook(Box::new(|_| {}));
    let result = panic::catch_unwind(panic::AssertUnwindSafe(f));
    panic::set_hook(hook);
    let payload = result.err()?;
    Some(match payload.downcast_ref::<&str>() {
        Some(s) => s.to_string(),
        None => payload
            .downcast_ref::<String>()
            .cloned()
            .unwrap_or_default(),
    })
}

fn main() {
    // (1) "<=" instead of "<": less(a, a) is true, so equal keys look "less".
    let input: Vec<Pair> = [3, 1, 2, 1, 3, 2, 1, 2, 3, 1]
        .into_iter()
        .zip(b'a'..)
        .map(|(key, tag)| Pair { key, tag })
        .collect();
    let lt = |a: &Pair, b: &Pair| a.key < b.key;
    let le = |a: &Pair, b: &Pair| a.key <= b.key;
    let cmp_lt = |a: &Pair, b: &Pair| a.key.cmp(&b.key);
    let cmp_le = |a: &Pair, b: &Pair| if a.key <= b.key { Less } else { Greater }; // never Equal
    println!(
        "(1) comparator says \"less\" for equal keys; input: {}",
        join(&input, pair_str)
    );
    println!(
        "    less(1b, 1b) = {} -- irreflexivity is broken",
        le(&input[1], &input[1])
    );
    let row1 = |label: &str, sort_it: &dyn Fn(&mut [Pair], &mut usize)| {
        let (mut s, mut calls) = (input.clone(), 0);
        sort_it(&mut s, &mut calls);
        let (sorted, stable) = sorted_and_stable(&s);
        println!(
            "    {:<27} {}  sorted? {:<5} stable? {:<5} calls {}",
            label,
            join(&s, pair_str),
            sorted,
            stable,
            calls
        );
    };
    row1("insertion, a < b", &|s, c| {
        insertion(s, |a, b| {
            *c += 1;
            lt(a, b)
        })
    });
    row1("insertion, a <= b", &|s, c| {
        insertion(s, |a, b| {
            *c += 1;
            le(a, b)
        })
    });
    row1("sort_by, Ord::cmp", &|s, c| {
        s.sort_by(|a, b| {
            *c += 1;
            cmp_lt(a, b)
        })
    });
    row1("sort_by, <= style", &|s, c| {
        s.sort_by(|a, b| {
            *c += 1;
            cmp_le(a, b)
        })
    });
    println!();

    // (2) "return a - b" overflows: MAX - (-1) wraps to a negative number.
    let ints: [i64; 16] = [
        5,
        i64::MAX,
        -3,
        i64::MIN + 1,
        0,
        i64::MAX - 2,
        7,
        i64::MIN,
        -8,
        i64::MAX - 1,
        2,
        i64::MIN + 2,
        1,
        -1,
        9,
        4,
    ];
    println!(
        "(2) cmp = a - b on {} ints near the int64 limits",
        ints.len()
    );
    println!(
        "    cmp(MAX, -1) = {}: a - b claims MAX < -1",
        i64::MAX.wrapping_sub(-1)
    );
    let row2 = |label: &str, sort_it: &dyn Fn(&mut [i64])| {
        let mut s = ints;
        sort_it(&mut s);
        println!(
            "    {:<24} sorted? {:<5} {}",
            label,
            s.is_sorted(),
            join(&s, name)
        );
    };
    row2("insertion, a - b", &|s| {
        insertion(s, |a, b| subtract(a, b) == Less)
    });
    row2("sort_unstable_by, a - b", &|s| s.sort_unstable_by(subtract));
    row2("sort_unstable_by, cmp", &|s| s.sort_unstable_by(i64::cmp));
    // At len 16 <= 20 sort_unstable_by IS insertion sort, so it makes the same
    // comparisons and the same mistakes as the first row (Go's SortFunc, which
    // is pdqsort above 12, got this particular input sorted by luck).
    println!();

    // (3) NaN under a naive comparator: NaN is "equal" to 1 and to 9, but 1 is
    // not equal to 9 -- incomparability is not transitive.
    // `fl.sort()` does not compile: error[E0277]: the trait bound `f64: Ord` is
    // not satisfied. Rust makes you choose what NaN means.
    let nan = f64::NAN;
    let fl = [3.0, nan, 1.0, 4.0, -nan, 1.5, 9.0, 2.0, nan, 6.0, 5.0, 0.0];
    println!(
        "(3) floats with 3 NaNs (one with the sign bit set): {}",
        join(&fl, float_str)
    );
    let row3 = |label: &str, sort_it: &dyn Fn(&mut [f64])| {
        let mut s = fl;
        match try_it(|| sort_it(&mut s)) {
            None => println!(
                "    {:<27} numbers in order? {:<5} {}",
                label,
                numbers_in_order(&s),
                join(&s, float_str)
            ),
            Some(msg) => println!("    {label:<27} PANICKED: {msg}"),
        }
    };
    row3("sort_by, unwrap_or(Equal)", &|s| s.sort_by(naive));
    row3("sort_by(f64::total_cmp)", &|s| s.sort_by(f64::total_cmp));
    row3("sort_by, partial_cmp.unwrap", &|s| {
        s.sort_by(|a, b| a.partial_cmp(b).unwrap())
    });
    println!();

    // (4) The same three comparators on 1000 random inputs per size, past the
    // insertion-sort cutoff (20 for both sort_unstable_by and sort_by).
    // Each trial draws an input for the comparator under test and runs both
    // sorts on copies of it.
    let mut r = Pcg::new(12, 11);
    println!("(4) 1000 random inputs per n: unsorted results / panics");
    println!(
        "    {:<6} {:>4}  {:<17} {:<7}",
        "cmp", "n", "sort_unstable_by", "sort_by"
    );
    let mut messages = std::collections::BTreeMap::new(); // panic message -> count
    for c in ["<=", "a - b", "naive"] {
        for n in [20, 21, 32, 64, 200] {
            let (mut bad, mut pan) = ([0; 2], [0; 2]);
            for _ in 0..1000 {
                let (mut ps, mut is, mut fs) = (Vec::new(), Vec::new(), Vec::new());
                for i in 0..n {
                    match c {
                        "<=" => ps.push(Pair {
                            key: r.int_n(4),
                            tag: i as u8,
                        }),
                        // a third each: near MIN, near MAX, small
                        "a - b" => is.push(match r.int_n(3) {
                            0 => i64::MIN + r.int_n(50) as i64,
                            1 => i64::MAX - r.int_n(50) as i64,
                            _ => r.int_n(100) as i64 - 50,
                        }),
                        // one in 8 is NaN
                        _ => {
                            let v = r.int_n(1000) as f64;
                            fs.push(if r.int_n(8) == 0 { nan } else { v });
                        }
                    }
                }
                for (k, stable) in [false, true].into_iter().enumerate() {
                    let (p, ok) = match c {
                        "<=" => {
                            let mut s = ps.clone();
                            let p = try_it(|| sort_with(&mut s, cmp_le, stable));
                            (p, sorted_and_stable(&s).0)
                        }
                        "a - b" => {
                            let mut s = is.clone();
                            let p = try_it(|| sort_with(&mut s, subtract, stable));
                            (p, s.is_sorted())
                        }
                        _ => {
                            let mut s = fs.clone();
                            let p = try_it(|| sort_with(&mut s, naive, stable));
                            (p, numbers_in_order(&s))
                        }
                    };
                    match p {
                        Some(msg) => {
                            pan[k] += 1;
                            *messages.entry(msg).or_insert(0) += 1;
                        }
                        None if !ok => bad[k] += 1,
                        None => {}
                    }
                }
            }
            println!(
                "    {:<6} {:>4}  {:>4} / {:<10} {:>4} / {}",
                c, n, bad[0], pan[0], bad[1], pan[1]
            );
        }
    }
    for (msg, count) in &messages {
        println!("    {count} panics, with the message: \"{msg}\"");
    }
}
```

**Rust output:**

```
(1) comparator says "less" for equal keys; input: 3a 1b 2c 1d 3e 2f 1g 2h 3i 1j
    less(1b, 1b) = true -- irreflexivity is broken
    insertion, a < b            1b 1d 1g 1j 2c 2f 2h 3a 3e 3i  sorted? true  stable? true  calls 26
    insertion, a <= b           1j 1g 1d 1b 2h 2f 2c 3i 3e 3a  sorted? true  stable? false calls 35
    sort_by, Ord::cmp           1b 1d 1g 1j 2c 2f 2h 3a 3e 3i  sorted? true  stable? true  calls 26
    sort_by, <= style           1j 1g 1d 1b 2h 2f 2c 3i 3e 3a  sorted? true  stable? false calls 35

(2) cmp = a - b on 16 ints near the int64 limits
    cmp(MAX, -1) = -9223372036854775808: a - b claims MAX < -1
    insertion, a - b         sorted? false 5 MAX-0 MIN+1 -3 0 7 MAX-2 MAX-1 MIN+0 MIN+2 -8 -1 1 2 4 9
    sort_unstable_by, a - b  sorted? false 5 MAX-0 MIN+1 -3 0 7 MAX-2 MAX-1 MIN+0 MIN+2 -8 -1 1 2 4 9
    sort_unstable_by, cmp    sorted? true  MIN+0 MIN+1 MIN+2 -8 -3 -1 0 1 2 4 5 7 9 MAX-2 MAX-1 MAX-0

(3) floats with 3 NaNs (one with the sign bit set): 3 NaN 1 4 -NaN 1.5 9 2 NaN 6 5 0
    sort_by, unwrap_or(Equal)   numbers in order? false 3 NaN 1 4 -NaN 1.5 2 9 NaN 0 5 6
    sort_by(f64::total_cmp)     numbers in order? true  -NaN 0 1 1.5 2 3 4 5 6 9 NaN NaN
    sort_by, partial_cmp.unwrap PANICKED: called `Option::unwrap()` on a `None` value

(4) 1000 random inputs per n: unsorted results / panics
    cmp       n  sort_unstable_by  sort_by
    <=       20     0 / 0             0 / 0
    <=       21     0 / 0             0 / 0
    <=       32     0 / 0             0 / 0
    <=       64     0 / 0             0 / 0
    <=      200     0 / 0             0 / 0
    a - b    20  1000 / 0          1000 / 0
    a - b    21   412 / 588         315 / 685
    a - b    32   329 / 671         245 / 755
    a - b    64   989 / 0            67 / 933
    a - b   200   999 / 0           999 / 0
    naive    20   900 / 0           900 / 0
    naive    21   177 / 701          68 / 840
    naive    32    99 / 871          29 / 945
    naive    64   445 / 492           3 / 996
    naive   200   865 / 47          744 / 59
    8583 panics, with the message: "user-provided comparison function does not correctly implement a total order"
```

> **Go vs Rust:** Identical where both run the same algorithm: section (1) (26 vs 35 calls — `SortStableFunc` and `sort_by` are both plain insertion sort at n = 10), the naive-NaN row, and the n = 20 rows of (4), which equal Go's `SortStableFunc` column (1000 and 900) because the PCG port feeds both programs the same inputs. Above 20 the two diverge by design: Go never panicked and returned unsorted output silently, while Rust panicked in 8,583 of 30,000 sorts with "does not correctly implement a total order". That check sits only in the small-sort's `bidirectional_merge` (`core/src/slice/sort/shared/smallsort.rs`, read in the source), so it fires at n = 21–64 and rarely at 200: it catches some bad comparators, not all, and `<=` never set it off. A caught panic unwinds (it is not an abort), so `catch_unwind` is safe here. Rust also refuses `fl.sort()` on `f64` at compile time; `total_cmp` puts `-NaN` first and `NaN` last, where Go puts every NaN first.

**Complexity:** no change — a broken comparator breaks correctness, not cost. Measured: `<=` reverses every group of ties (35 calls instead of 26) yet never left an output unsorted in 10,000 trials; `a - b` left 95–100% of random inputs unsorted (while one fixed input happened to come out sorted under `SortFunc`); the naive NaN comparator 72–100%. `slices.Sort` and `cmp.Compare` put the NaNs first and the numbers in order. Go never panicked: it returns garbage silently.

---

> Next tier: [🔴 hard](3-hard.md) — the standard library's insertion-sort cutoff, caught by counting,
> what it does with presorted input, Shell sort, expensive writes, and the capstone.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
