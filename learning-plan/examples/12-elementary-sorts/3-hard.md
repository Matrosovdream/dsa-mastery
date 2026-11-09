# Step 12 — Elementary Sorts · 🔴 Hard

Examples **12–16**: the standard library's insertion-sort cutoff found by counting comparisons, what
the stdlib does with presorted input, Shell sort past the adjacent-swap barrier, sorting when writes
are expensive, and the capstone.

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

> ← Back to [🟡 medium](2-medium.md) · Index: [README.md](README.md) · Progress: [PROGRESS.md](PROGRESS.md)

---

## 12. Catching the stdlib's insertion-sort cutoff

`🔴 hard` · *pdqsort's small-n fallback*

Every fast library sort hands short ranges to insertion sort, and you can find out where without reading a
line of library code: give the sort a comparator that counts its own calls and compare that count with your own
insertion sort on the same permutation. Insertion sort's count (the inversions plus the elements that stop early)
is so specific that a different algorithm matches it on all 100 of 100 random permutations only by being that
algorithm — so the length where "100 of 100" breaks is the cutoff, and below it the library is Θ(n²) insertion
sort on purpose. The consequence is the practical half: below the cutoff the *unstable* sort is stable by
accident, so a small test fixture cannot catch code that wrongly relies on stability.

**Steps:**

1. Write the swap version of insertion sort with a comparison counter — the exact loop the Go stdlib uses.
2. For n = 2…40, run 100 seeded permutations through it, `slices.SortFunc` and `slices.SortStableFunc` with a
   counting `cmp`, and record how often the counts agree; detect each cutoff as "the last n with 100 of 100".
3. Sort records with only 3 distinct keys and watch for the first n at which `SortFunc` reorders equal keys.

```go
package main

import (
	"cmp"
	"fmt"
	"math/rand/v2"
	"slices"
)

// The question: below which length does the standard library stop being
// pdqsort (or merge sort) and simply run insertion sort? We do not read the
// answer off a blog post -- we catch it with nothing but a counting comparator.
//
// The trick: insertion sort makes a very particular number of comparisons on a
// given permutation (inversions + the elements that stop early). A different
// algorithm -- quicksort with median-of-three, a merge -- almost never makes
// exactly the same number on every one of 100 random permutations. So "the
// stdlib's count equals insertion sort's count on 100 of 100 trials" is strong
// evidence that, at that n, the stdlib IS insertion sort.

// insertionCount is the swap version of insertion sort -- the same loop as Go's
// own insertionSortCmpFunc (src/slices/zsortanyfunc.go):
//
//	for j := i; j > a && cmp(data[j], data[j-1]) < 0; j-- { swap }
//
// It returns the number of comparisons it made.
func insertionCount(s []int) int {
	c := 0
	for i := 1; i < len(s); i++ {
		for j := i; j > 0; j-- {
			c++ // one call of the comparison, as the stdlib would make it
			if s[j] >= s[j-1] {
				break // not less: s[j] has arrived
			}
			s[j], s[j-1] = s[j-1], s[j]
		}
	}
	return c
}

// countWith runs a stdlib sort with a comparator that counts its own calls.
func countWith(sort func([]int, func(a, b int) int), s []int) int {
	c := 0
	sort(s, func(a, b int) int {
		c++
		return cmp.Compare(a, b)
	})
	return c
}

// The two sorts under test, as plain functions over []int.
func sortFunc(s []int, f func(a, b int) int)       { slices.SortFunc(s, f) }
func sortStableFunc(s []int, f func(a, b int) int) { slices.SortStableFunc(s, f) }

// perm is a Fisher-Yates shuffle of 0..n-1 driven by r.IntN, written out by
// hand (not r.Perm) so the Rust twin can reproduce the exact permutations.
func perm(r *rand.Rand, n int) []int {
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

// cutoff scans the match table: the last n before the first n whose match is
// below 100%. Above the cutoff a few coincidental matches are expected, so the
// detector insists on 100 of 100.
func cutoff(match map[int]int, lo, hi, trials int) (cut, firstMissN, firstMiss, bestAbove int) {
	cut = hi
	for n := lo; n <= hi; n++ {
		if match[n] < trials {
			cut, firstMissN, firstMiss = n-1, n, match[n]
			break
		}
	}
	for n := cut + 1; n <= hi; n++ {
		bestAbove = max(bestAbove, match[n])
	}
	return
}

type rec struct{ key, tag int } // tag = input position, to see reordering

// reorders reports whether two equal keys came out in a different order than
// they went in -- i.e. whether this run of the sort was unstable.
func reorders(out []rec) bool {
	for i := 1; i < len(out); i++ {
		if out[i].key == out[i-1].key && out[i].tag < out[i-1].tag {
			return true
		}
	}
	return false
}

func main() {
	const lo, hi, trials = 2, 40, 100
	r := rand.New(rand.NewPCG(1, 2))

	fmt.Printf("Table 1: trials (of %d random permutations) where the stdlib's comparator\n", trials)
	fmt.Println("         calls equal our insertion sort's on the same permutation")
	fmt.Println("   n   slices.SortFunc   slices.SortStableFunc")
	mFunc, mStable := map[int]int{}, map[int]int{}
	for n := lo; n <= hi; n++ {
		for t := 0; t < trials; t++ {
			p := perm(r, n)
			want := insertionCount(slices.Clone(p))
			if countWith(sortFunc, slices.Clone(p)) == want {
				mFunc[n]++
			}
			if countWith(sortStableFunc, slices.Clone(p)) == want {
				mStable[n]++
			}
		}
		fmt.Printf("%4d   %15d   %21d\n", n, mFunc[n], mStable[n])
	}

	fmt.Println()
	report := func(name string, m map[int]int, constant int, source string) {
		cut, missN, miss, best := cutoff(m, lo, hi, trials)
		fmt.Printf("%-22s insertion sort for n <= %d (n = %d: %d/%d match; at most %d for any n > %d)\n",
			name, cut, missN, miss, trials, best, cut)
		verdict := "agrees"
		if cut != constant {
			verdict = "DISAGREES"
		}
		fmt.Printf("%-22s source: %s -> %s\n", "", source, verdict)
	}
	report("slices.SortFunc", mFunc, 12,
		"const maxInsertion = 12, src/slices/zsortanyfunc.go")
	report("slices.SortStableFunc", mStable, 20,
		"blockSize := 20, src/slices/zsortanyfunc.go")

	// The consequence. Insertion sort with a strict "<" never moves an element
	// past an equal one, so below its cutoff the "unstable" SortFunc is stable
	// by accident. Records with only 3 distinct keys make ties certain.
	const hi2, trials2 = 64, 200
	r2 := rand.New(rand.NewPCG(3, 4))
	unstable, stable := map[int]int{}, map[int]int{}
	byKey := func(a, b rec) int { return cmp.Compare(a.key, b.key) }
	for n := 2; n <= hi2; n++ {
		for t := 0; t < trials2; t++ {
			in := make([]rec, n)
			for i := range in {
				in[i] = rec{key: r2.IntN(3), tag: i}
			}
			a, b := slices.Clone(in), slices.Clone(in)
			slices.SortFunc(a, byKey)
			slices.SortStableFunc(b, byKey)
			if reorders(a) {
				unstable[n]++
			}
			if reorders(b) {
				stable[n]++
			}
		}
	}
	fmt.Println()
	fmt.Printf("Table 2: trials (of %d) where equal keys came out reordered; keys in [0, 3)\n", trials2)
	fmt.Println("   n   slices.SortFunc   slices.SortStableFunc")
	for _, n := range []int{4, 8, 12, 13, 14, 16, 20, 21, 24, 32, 33, 40, 48, 64} {
		fmt.Printf("%4d   %15d   %21d\n", n, unstable[n], stable[n])
	}
	describe := func(m map[int]int) string {
		if n := firstN(m, hi2); n <= hi2 {
			return fmt.Sprintf("n = %d", n)
		}
		return fmt.Sprintf("none up to n = %d", hi2)
	}
	fmt.Printf("first reordered tie: slices.SortFunc %s; slices.SortStableFunc %s\n",
		describe(unstable), describe(stable))
	fmt.Printf("no fixture of <= %d records saw SortFunc reorder a tie -- such tests cannot catch it\n",
		firstN(unstable, hi2)-1)
}

// firstN is the first n with a non-zero count (hi+1 if none).
func firstN(m map[int]int, hi int) int {
	for n := 2; n <= hi; n++ {
		if m[n] > 0 {
			return n
		}
	}
	return hi + 1
}
```

**Output:**

```
Table 1: trials (of 100 random permutations) where the stdlib's comparator
         calls equal our insertion sort's on the same permutation
   n   slices.SortFunc   slices.SortStableFunc
   2               100                     100
   3               100                     100
   4               100                     100
   5               100                     100
   6               100                     100
   7               100                     100
   8               100                     100
   9               100                     100
  10               100                     100
  11               100                     100
  12               100                     100
  13                 3                     100
  14                 4                     100
  15                 1                     100
  16                 2                     100
  17                 1                     100
  18                 2                     100
  19                 1                     100
  20                 0                     100
  21                 0                       7
  22                 0                       6
  23                 0                       0
  24                 0                       1
  25                 0                       0
  26                 0                       0
  27                 0                       0
  28                 0                       0
  29                 0                       0
  30                 0                       0
  31                 0                       0
  32                 0                       0
  33                 0                       0
  34                 0                       0
  35                 0                       0
  36                 0                       0
  37                 0                       0
  38                 0                       0
  39                 0                       0
  40                 0                       0

slices.SortFunc        insertion sort for n <= 12 (n = 13: 3/100 match; at most 4 for any n > 12)
                       source: const maxInsertion = 12, src/slices/zsortanyfunc.go -> agrees
slices.SortStableFunc  insertion sort for n <= 20 (n = 21: 7/100 match; at most 7 for any n > 20)
                       source: blockSize := 20, src/slices/zsortanyfunc.go -> agrees

Table 2: trials (of 200) where equal keys came out reordered; keys in [0, 3)
   n   slices.SortFunc   slices.SortStableFunc
   4                 0                       0
   8                 0                       0
  12                 0                       0
  13               200                       0
  14               200                       0
  16               200                       0
  20               200                       0
  21               200                       0
  24               200                       0
  32               200                       0
  33               200                       0
  40               200                       0
  48               200                       0
  64               200                       0
first reordered tie: slices.SortFunc n = 13; slices.SortStableFunc none up to n = 64
no fixture of <= 12 records saw SortFunc reorder a tie -- such tests cannot catch it
```

**Rust:**

```rust
// The question: below which length does the standard library stop being a
// clever sort and simply run insertion sort? We catch the answer with nothing
// but a counting comparator.
//
// The trick: insertion sort makes a very particular number of comparisons on a
// given permutation (inversions + the elements that stop early). A different
// algorithm -- quicksort, a merge, a sorting network -- almost never makes
// exactly the same number on every one of 100 random permutations. So "the
// std count equals insertion sort's count on 100 of 100 trials" is strong
// evidence that, at that n, std IS insertion sort.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so this program sees exactly the permutations the Go program sees.
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

/// The swap version of insertion sort -- the same comparison sequence as Go's
/// insertionSortCmpFunc AND as Rust's own `insert_tail` (which shifts through a
/// hole instead of swapping, but asks `is_less` the same questions in the same
/// order). Returns the number of comparisons.
fn insertion_count(s: &mut [u64]) -> usize {
    let mut c = 0;
    for i in 1..s.len() {
        for j in (1..=i).rev() {
            c += 1; // one call of the comparison, as std would make it
            if s[j] >= s[j - 1] {
                break; // not less: s[j] has arrived
            }
            s.swap(j, j - 1);
        }
    }
    c
}

// A comparator that counts its own calls. The closure borrows `c` mutably;
// `sort_*_by` takes `FnMut`, so this needs no Cell and no global.
fn count_unstable(mut v: Vec<u64>) -> usize {
    let mut c = 0;
    v.sort_unstable_by(|a, b| {
        c += 1;
        a.cmp(b)
    });
    c
}

fn count_stable(mut v: Vec<u64>) -> usize {
    let mut c = 0;
    v.sort_by(|a, b| {
        c += 1;
        a.cmp(b)
    });
    c
}

/// Fisher-Yates on 0..n-1 driven by int_n -- the same draws as the Go program.
fn perm(r: &mut Pcg, n: usize) -> Vec<u64> {
    let mut p: Vec<u64> = (0..n as u64).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i as u64 + 1) as usize;
        p.swap(i, j);
    }
    p
}

struct Cutoff {
    cut: usize,
    miss_n: usize,
    miss: usize,
    best_above: usize,
}

/// The last n before the first n whose match is below 100%. Above the cutoff a
/// few coincidental matches are expected, so the detector insists on all trials.
/// `matches[k]` belongs to n = lo + k.
fn cutoff(matches: &[usize], lo: usize, trials: usize) -> Cutoff {
    let first_miss = matches.iter().position(|&m| m < trials);
    let k = first_miss.unwrap_or(matches.len());
    Cutoff {
        cut: lo + k - 1,
        miss_n: lo + k,
        miss: matches.get(k).copied().unwrap_or(trials),
        best_above: matches.iter().skip(k).copied().max().unwrap_or(0),
    }
}

/// Did two equal keys come out in a different order than they went in?
/// (key, tag) with tag = input position. Generic so the same check serves the
/// 16-byte and the 8-byte record below.
fn reorders<K: Eq, T: Ord>(out: &[(K, T)]) -> bool {
    out.windows(2).any(|w| w[0].0 == w[1].0 && w[1].1 < w[0].1)
}

fn main() {
    const LO: usize = 2;
    const HI: usize = 40;
    const TRIALS: usize = 100;
    let mut r = Pcg::new(1, 2);

    println!("Table 1: trials (of {TRIALS} random permutations) where std's comparator");
    println!("         calls equal our insertion sort's on the same permutation");
    println!("   n   sort_unstable_by           sort_by");
    let (mut m_unstable, mut m_stable) = (Vec::new(), Vec::new());
    for n in LO..=HI {
        let (mut a, mut b) = (0, 0);
        for _ in 0..TRIALS {
            let p = perm(&mut r, n);
            let want = insertion_count(&mut p.clone());
            a += usize::from(count_unstable(p.clone()) == want);
            b += usize::from(count_stable(p) == want);
        }
        println!("{n:4}   {a:16}   {b:15}");
        m_unstable.push(a);
        m_stable.push(b);
    }

    println!();
    let report = |name: &str, m: &[usize], constant: usize, source: &str| {
        let c = cutoff(m, LO, TRIALS);
        println!(
            "{name:<22} insertion sort for n <= {} (n = {}: {}/{TRIALS} match; at most {} for any n > {})",
            c.cut, c.miss_n, c.miss, c.best_above, c.cut
        );
        let verdict = if c.cut == constant {
            "agrees"
        } else {
            "DISAGREES"
        };
        println!("{:<22} source: {source} -> {verdict}", "");
    };
    report(
        "sort_unstable_by",
        &m_unstable,
        20,
        "MAX_LEN_ALWAYS_INSERTION_SORT = 20, core/src/slice/sort/unstable/mod.rs",
    );
    report(
        "sort_by",
        &m_stable,
        20,
        "MAX_LEN_ALWAYS_INSERTION_SORT = 20, core/src/slice/sort/stable/mod.rs",
    );

    // The consequence. Insertion sort with a strict "<" never moves an element
    // past an equal one, so below its cutoff the "unstable" sort is stable by
    // accident. Records with only 3 distinct keys make ties certain.
    // Rust adds a twist Go does not have: above 20, what sort_unstable_by does
    // on a small slice depends on the element TYPE (see the note), so we sort
    // the same records as a 16-byte (u64, u64) and as an 8-byte (u32, u32).
    const HI2: usize = 64;
    const TRIALS2: usize = 200;
    let mut r2 = Pcg::new(3, 4);
    // counts[n] = [unstable 16-byte, stable 16-byte, unstable 8-byte]
    let mut counts = vec![[0usize; 3]; HI2 + 1];
    for (n, row) in counts.iter_mut().enumerate().skip(2) {
        for _ in 0..TRIALS2 {
            let input: Vec<(u64, u64)> = (0..n as u64).map(|i| (r2.int_n(3), i)).collect();
            let mut a = input.clone();
            let mut b = input.clone();
            let mut c: Vec<(u32, u32)> = input.iter().map(|&(k, t)| (k as u32, t as u32)).collect();
            a.sort_unstable_by_key(|x| x.0); // compare the key only (the tag rides along)
            b.sort_by_key(|x| x.0);
            c.sort_unstable_by_key(|x| x.0);
            row[0] += usize::from(reorders(&a));
            row[1] += usize::from(reorders(&b));
            row[2] += usize::from(reorders(&c));
        }
    }
    println!();
    println!("Table 2: trials (of {TRIALS2}) where equal keys came out reordered; keys in [0, 3)");
    println!("         record (key, tag) as 16 bytes (u64, u64) and as 8 bytes (u32, u32)");
    println!("   n   sort_unstable_by 16B   sort_by 16B   sort_unstable_by 8B");
    for n in [4, 8, 12, 13, 14, 16, 20, 21, 24, 32, 33, 40, 48, 64] {
        let [a, b, c] = counts[n];
        println!("{n:4}   {a:20}   {b:11}   {c:19}");
    }
    let first = |col: usize| (2..=HI2).find(|&n| counts[n][col] > 0);
    let describe = |col: usize| match first(col) {
        Some(n) => format!("n = {n}"),
        None => format!("none up to n = {HI2}"),
    };
    println!(
        "first reordered tie: sort_unstable_by 16B {}; sort_by {}; sort_unstable_by 8B {}",
        describe(0),
        describe(1),
        describe(2)
    );
    let safe = (0..3)
        .filter(|&c| c != 1)
        .map(|c| first(c).unwrap_or(HI2 + 1))
        .min()
        .unwrap_or(HI2 + 1)
        - 1;
    println!(
        "no fixture of <= {safe} records saw sort_unstable_by reorder a tie -- such tests cannot catch it"
    );
}
```

**Rust output:**

```
Table 1: trials (of 100 random permutations) where std's comparator
         calls equal our insertion sort's on the same permutation
   n   sort_unstable_by           sort_by
   2                100               100
   3                100               100
   4                100               100
   5                100               100
   6                100               100
   7                100               100
   8                100               100
   9                100               100
  10                100               100
  11                100               100
  12                100               100
  13                100               100
  14                100               100
  15                100               100
  16                100               100
  17                100               100
  18                100               100
  19                100               100
  20                100               100
  21                  0                 0
  22                  1                 0
  23                  0                 0
  24                  0                 0
  25                  0                 0
  26                  0                 0
  27                  0                 0
  28                  0                 0
  29                  0                 0
  30                  0                 0
  31                  0                 0
  32                  0                 0
  33                  0                 0
  34                  0                 0
  35                  0                 0
  36                  0                 0
  37                  0                 0
  38                  0                 0
  39                  0                 0
  40                  0                 0

sort_unstable_by       insertion sort for n <= 20 (n = 21: 0/100 match; at most 1 for any n > 20)
                       source: MAX_LEN_ALWAYS_INSERTION_SORT = 20, core/src/slice/sort/unstable/mod.rs -> agrees
sort_by                insertion sort for n <= 20 (n = 21: 0/100 match; at most 0 for any n > 20)
                       source: MAX_LEN_ALWAYS_INSERTION_SORT = 20, core/src/slice/sort/stable/mod.rs -> agrees

Table 2: trials (of 200) where equal keys came out reordered; keys in [0, 3)
         record (key, tag) as 16 bytes (u64, u64) and as 8 bytes (u32, u32)
   n   sort_unstable_by 16B   sort_by 16B   sort_unstable_by 8B
   4                      0             0                     0
   8                      0             0                     0
  12                      0             0                     0
  13                      0             0                     0
  14                      0             0                     0
  16                      0             0                     0
  20                      0             0                     0
  21                      0             0                   200
  24                      0             0                   200
  32                      0             0                   200
  33                    177             0                   200
  40                    200             0                   200
  48                    200             0                   200
  64                    200             0                   200
first reordered tie: sort_unstable_by 16B n = 33; sort_by none up to n = 64; sort_unstable_by 8B n = 21
no fixture of <= 20 records saw sort_unstable_by reorder a tie -- such tests cannot catch it
```

> **Go vs Rust:** Same permutations (PCG port), different library, so the tables differ by design: Rust's `sort_unstable_by` and `sort_by` both match insertion sort on 100/100 up to n = 20 and on at most 1/100 above it — `MAX_LEN_ALWAYS_INSERTION_SORT = 20` checks the length of the **whole slice** once (`core/src/slice/sort/{stable,unstable}/mod.rs`), where Go's 12 is re-checked on every pdqsort sub-range. Rust's `insert_tail` shifts through a hole instead of swapping but asks `is_less` the same questions, which is why the counts can match at all.
> The stability column depends on the element **type** in Rust, which Go has no analogue for: the 16-byte `(u64, u64)` records first reordered a tie at n = 33, the 8-byte `(u32, u32)` ones at n = 21. Read in the source, not run: above 20 ipnsort's small-sort (`shared/smallsort.rs`, threshold 32) uses sorting networks (`sort9_optimal`/`sort13_optimal`, unstable) for `Copy` types of at most 8 bytes and `sort8_stable` + `bidirectional_merge` (stable pieces) for larger ones.

**Complexity:** below its cutoff the stdlib *is* Θ(n²) insertion sort, and that is the right choice for a handful of elements · measured with nothing but a counting comparator: Go `slices.SortFunc` matches insertion sort on 100/100 permutations up to n = 12 (`maxInsertion`, and pdqsort applies it to every sub-range, not only the top call) and `slices.SortStableFunc` up to n = 20 (its blocks of 20) · `SortFunc` first reordered equal keys at n = 13, in 200 of 200 trials — and never at n ≤ 12. Rust's cutoff is 20 for both sorts (see the Rust twin).

---

## 13. What the stdlib does with presorted input

`🔴 hard` · *Adaptivity inside pdqsort, ipnsort and driftsort*

Insertion sort costs Θ(n + I), so it is linear on sorted input and on anything a few inversions away from it.
Library sorts are Θ(n log n) in general but carry shortcuts for presorted data, and a counting comparator shows
exactly where each shortcut stops: Go's pdqsort reverses a range whose pivot samples all say "descending" and
tries `partialInsertionSort` (at most 5 fix-up rounds) when they all say "ascending"; its stable sort has no
presorted shortcut beyond its insertion-sorted blocks of 20 and an in-place merge that is cheap on ordered
data. The rows that teach are the near misses — one element out of place, four versus five misplaced pairs, a
sorted run with a random tail, a descending input with duplicates — because that is where "the stdlib is adaptive"
stops being true and insertion sort keeps its edge.

**Steps:**

1. Build ten inputs of n = 10,000: exact patterns, near misses and two seeded random ones (swap positions, tail).
2. Count comparator calls for a shift insertion sort (its shifts give I for free), `slices.SortFunc` and
   `slices.SortStableFunc`; mark every count ≤ 3n as linear.
3. Time the same sorts as a supplement, restoring the pristine input before every run and subtracting a copy-only loop.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"strings"
	"time"
)

// Insertion sort is linear on sorted input. Is the standard library? A counting
// comparator answers it exactly, and the interesting rows are the NEAR misses:
// pdqsort has two shortcuts for presorted data, and each has a sharp edge.
//
//   1. choosePivot samples a few elements; if every sample comparison says
//      "descending" it reverses the whole range first (decreasingHint).
//   2. if the samples look ascending, partialInsertionSort makes at most 5
//      rounds (maxSteps = 5) of "scan to the next out-of-order pair, fix it";
//      it reports success only when a scan reaches the end. So 4 misplaced
//      pairs are fixed AND verified in round 5, while the 5th pair is fixed in
//      round 5 and never verified: pdqsort falls through to partitioning.
//
// We count, for each input: our insertion sort, slices.SortFunc,
// slices.SortStableFunc -- and call a count "O(n)" when it is <= 3n.

const n = 10_000

// insertion is the shift version; it returns (comparisons, shifts).
// shifts == inversions exactly (example 6), so this also measures I.
func insertion(s []int) (comps, shifts int) {
	for i := 1; i < len(s); i++ {
		x := s[i]
		j := i
		for j > 0 {
			comps++
			if s[j-1] <= x {
				break
			}
			s[j] = s[j-1] // shift one place right
			shifts++
			j--
		}
		s[j] = x
	}
	return
}

func countSortFunc(s []int) int {
	c := 0
	slices.SortFunc(s, func(a, b int) int { c++; return cmp.Compare(a, b) })
	return c
}

func countStable(s []int) int {
	c := 0
	slices.SortStableFunc(s, func(a, b int) int { c++; return cmp.Compare(a, b) })
	return c
}

type input struct {
	name string
	data []int
}

func inputs() []input {
	r := rand.New(rand.NewPCG(5, 6))
	asc := func() []int {
		s := make([]int, n)
		for i := range s {
			s[i] = i
		}
		return s
	}
	var in []input
	in = append(in, input{"ascending", asc()})

	s := asc()
	slices.Reverse(s)
	in = append(in, input{"strictly descending", s})

	s = make([]int, n) // n/2-1, n/2-1, ..., 1, 1, 0, 0
	for i := range s {
		s[i] = (n - 1 - i) / 2
	}
	in = append(in, input{"non-strictly descending", s})

	s = append([]int{n - 1}, asc()[:n-1]...) // the largest, moved to the front
	in = append(in, input{"last moved to front", s})

	s = append(asc()[1:], 0) // the smallest, moved to the end
	in = append(in, input{"first moved to end", s})

	// k swaps of adjacent pairs at random, non-overlapping positions:
	// exactly k inversions. 4 is the most partialInsertionSort can verify.
	for _, k := range []int{4, 5} {
		s = asc()
		used := map[int]bool{}
		for len(used) < k {
			p := r.IntN(n - 1)
			if used[p] || used[p-1] || used[p+1] {
				continue // keep the swapped pairs apart
			}
			used[p] = true
			s[p], s[p+1] = s[p+1], s[p]
		}
		in = append(in, input{fmt.Sprintf("%d adjacent swaps", k), s})
	}

	s = asc()[:n-100] // 9,900 sorted, then 100 random values appended
	for range 100 {
		s = append(s, r.IntN(n))
	}
	in = append(in, input{"sorted + 100 random", s})

	in = append(in, input{"all equal", make([]int, n)})

	s = make([]int, n) // 0, 1, ..., n/2-1, n/2-1, ..., 1, 0
	for i := range s {
		s[i] = min(i, n-1-i)
	}
	in = append(in, input{"organ pipe", s})
	return in
}

// linear marks a count as O(n) by the rule of this example: <= 3n.
func linear(c int) string {
	if c <= 3*n {
		return "*"
	}
	return " "
}

// timeSort reports the best of 3 measurements of the sort alone, in µs. The
// sort mutates its input, so EVERY iteration first copies the pristine input
// back (sorting an already-sorted slice would measure the wrong row); the
// same loop with the copy only is timed too and subtracted.
func timeSort(pristine []int, sort func([]int)) float64 {
	buf := make([]int, len(pristine))
	measure := func(reps int, withSort bool) time.Duration {
		start := time.Now()
		for range reps {
			copy(buf, pristine)
			if withSort {
				sort(buf)
			}
		}
		return time.Since(start)
	}
	reps := 1 // grow until one batch takes >= 5 ms
	for measure(reps, true) < 5*time.Millisecond {
		reps *= 2
	}
	best := time.Duration(1 << 62)
	for range 3 {
		best = min(best, measure(reps, true)-measure(reps, false))
	}
	return float64(best.Nanoseconds()) / float64(reps) / 1000
}

func main() {
	in := inputs()
	fmt.Printf("n = %d; n-1 = %d, n*log2(n) = %.0f; * = O(n) here, i.e. <= 3n = %d comparisons\n\n",
		n, n-1, n*math.Log2(n), 3*n)
	fmt.Printf("%-26s %10s %11s %11s %11s\n", "input", "I", "insertion", "SortFunc", "SortStable")
	fmt.Println(strings.Repeat("-", 73))
	notLinear := map[string][]string{} // inputs where a sort needed > 3n
	funcCount := map[string]int{}      // SortFunc's count per input, for the cliff line
	for _, x := range in {
		ic, shifts := insertion(slices.Clone(x.data))
		fc := countSortFunc(slices.Clone(x.data))
		sc := countStable(slices.Clone(x.data))
		funcCount[x.name] = fc
		fmt.Printf("%-26s %10d %10d%s %10d%s %10d%s\n", x.name, shifts,
			ic, linear(ic), fc, linear(fc), sc, linear(sc))
		for _, c := range []struct {
			name  string
			count int
		}{{"insertion", ic}, {"SortFunc", fc}, {"SortStable", sc}} {
			if c.count > 3*n {
				notLinear[c.name] = append(notLinear[c.name], x.name)
			}
		}
	}
	fmt.Println()
	for _, name := range []string{"insertion", "SortFunc", "SortStable"} {
		fmt.Printf("%-10s O(n) on %d of %d inputs; not on: %s\n", name,
			len(in)-len(notLinear[name]), len(in), strings.Join(notLinear[name], ", "))
	}

	a, b := funcCount["4 adjacent swaps"], funcCount["5 adjacent swaps"]
	ratio := float64(b) / float64(a)
	verdict := "no" // partialInsertionSort predicts a cliff; say what THIS run shows
	if ratio >= 1.5 {
		verdict = "a"
	}
	fmt.Printf("\n%s 4-vs-5 cliff here: SortFunc needs %d comparisons for 4 adjacent swaps, %d for 5 (x%.1f)\n",
		verdict, a, b, ratio)

	// One timing table, as a supplement to the counts. slices.Sort runs the
	// same pdqsort as SortFunc (both are generated from one template) but
	// compares ints with an inlined "<"; our insertion sort does too.
	fmt.Println()
	fmt.Println("time per sort, µs (best of 3, copy cost subtracted) -- varies by machine")
	fmt.Printf("%-26s %10s %11s %15s\n", "input", "insertion", "slices.Sort", "SortStableFunc")
	for _, x := range in {
		ti := timeSort(x.data, func(s []int) { insertion(s) })
		tf := timeSort(x.data, slices.Sort[[]int])
		ts := timeSort(x.data, func(s []int) { slices.SortStableFunc(s, cmp.Compare[int]) })
		fmt.Printf("%-26s %10.1f %11.1f %15.1f\n", x.name, ti, tf, ts)
	}
}
```

**Sample output:**

```
n = 10000; n-1 = 9999, n*log2(n) = 132877; * = O(n) here, i.e. <= 3n = 30000 comparisons

input                               I   insertion    SortFunc  SortStable
-------------------------------------------------------------------------
ascending                           0       9999*      10011*      12156*
strictly descending          49995000   49995000       10011*      98192 
non-strictly descending      49990000   49995000       37573       98192 
last moved to front              9999      19997*      20010*      12302*
first moved to end               9999      19997*      20010*      12247*
4 adjacent swaps                    4      10003*      10023*      12160*
5 adjacent swaps                    5      10004*      28941*      12160*
sorted + 100 random            549209     559208      292634       13862*
all equal                           0       9999*      10011*      12156*
organ pipe                   24995000   25004999      128181       71960 

insertion  O(n) on 6 of 10 inputs; not on: strictly descending, non-strictly descending, sorted + 100 random, organ pipe
SortFunc   O(n) on 7 of 10 inputs; not on: non-strictly descending, sorted + 100 random, organ pipe
SortStable O(n) on 7 of 10 inputs; not on: strictly descending, non-strictly descending, organ pipe

a 4-vs-5 cliff here: SortFunc needs 10023 comparisons for 4 adjacent swaps, 28941 for 5 (x2.9)

time per sort, µs (best of 3, copy cost subtracted) -- varies by machine
input                       insertion slices.Sort  SortStableFunc
ascending                         6.0         5.7            21.8
strictly descending           12802.0         7.1           180.5
non-strictly descending       13525.2        18.6           188.1
last moved to front               8.3        10.1            28.4
first moved to end                7.2        11.5            27.2
4 adjacent swaps                  5.7         5.6            21.3
5 adjacent swaps                  5.8        13.9            21.4
sorted + 100 random             146.1       175.4            43.5
all equal                         5.7         5.6            22.0
organ pipe                     6541.0        82.3           191.6
```

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// Insertion sort is linear on sorted input. Is std? A counting comparator
// answers it exactly, and the interesting rows are the NEAR misses. Rust's two
// sorts share one presorted shortcut, with a sharp edge of its own:
//
//   find_existing_run (core/src/slice/sort/shared/mod.rs) scans from the front
//   while the slice is ascending (a[i] >= a[i-1]) -- or, if the first pair
//   descends, while it is STRICTLY descending (a[i] < a[i-1]). ipnsort
//   (sort_unstable) only wins if that run is the whole slice; driftsort (sort)
//   keeps every run of at least ~sqrt(n) and merges them.
//
// We count, for each input: our insertion sort, sort_unstable_by, sort_by --
// and call a count "O(n)" when it is <= 3n.

const N: usize = 10_000;

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and IntN, so the random
// rows (swap positions, appended values) are exactly the Go program's.
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

/// The shift version; returns (comparisons, shifts).
/// shifts == inversions exactly (example 6), so this also measures I.
fn insertion(s: &mut [i64]) -> (usize, usize) {
    let (mut comps, mut shifts) = (0, 0);
    for i in 1..s.len() {
        let x = s[i]; // i64 is Copy: holding it leaves no "hole" to worry about
        let mut j = i;
        while j > 0 {
            comps += 1;
            if s[j - 1] <= x {
                break;
            }
            s[j] = s[j - 1]; // shift one place right
            shifts += 1;
            j -= 1;
        }
        s[j] = x;
    }
    (comps, shifts)
}

fn count_unstable(mut v: Vec<i64>) -> usize {
    let mut c = 0;
    v.sort_unstable_by(|a, b| {
        c += 1;
        a.cmp(b)
    });
    c
}

fn count_stable(mut v: Vec<i64>) -> usize {
    let mut c = 0;
    v.sort_by(|a, b| {
        c += 1;
        a.cmp(b)
    });
    c
}

fn inputs() -> Vec<(String, Vec<i64>)> {
    let mut r = Pcg::new(5, 6);
    let n = N as i64;
    let asc = || (0..n).collect::<Vec<i64>>();
    let mut v = vec![("ascending".to_string(), asc())];

    v.push(("strictly descending".into(), (0..n).rev().collect()));
    // n/2-1, n/2-1, ..., 1, 1, 0, 0
    v.push((
        "non-strictly descending".into(),
        (0..n).map(|i| (n - 1 - i) / 2).collect(),
    ));
    // the largest, moved to the front
    v.push((
        "last moved to front".into(),
        std::iter::once(n - 1).chain(0..n - 1).collect(),
    ));
    // the smallest, moved to the end
    v.push((
        "first moved to end".into(),
        (1..n).chain(std::iter::once(0)).collect(),
    ));

    // k swaps of adjacent pairs at random, non-overlapping positions:
    // exactly k inversions (4 and 5 are Go's interesting pair; same input here).
    for k in [4, 5] {
        let mut s = asc();
        let mut used = vec![false; N];
        let mut placed = 0;
        while placed < k {
            let p = r.int_n(N as u64 - 1) as usize;
            if used[p] || (p > 0 && used[p - 1]) || used[p + 1] {
                continue; // keep the swapped pairs apart
            }
            used[p] = true;
            placed += 1;
            s.swap(p, p + 1);
        }
        v.push((format!("{k} adjacent swaps"), s));
    }

    // 9,900 sorted, then 100 random values appended
    let mut s: Vec<i64> = (0..n - 100).collect();
    s.extend((0..100).map(|_| r.int_n(N as u64) as i64));
    v.push(("sorted + 100 random".into(), s));

    v.push(("all equal".into(), vec![0; N]));
    // 0, 1, ..., n/2-1, n/2-1, ..., 1, 0
    v.push((
        "organ pipe".into(),
        (0..n).map(|i| i.min(n - 1 - i)).collect(),
    ));
    v
}

/// The O(n) mark of this example: <= 3n comparisons.
fn linear(c: usize) -> &'static str {
    if c <= 3 * N { "*" } else { " " }
}

/// Best of 3 measurements of the sort alone, in µs. The sort mutates its
/// input, so EVERY iteration first copies the pristine input back (sorting an
/// already-sorted slice would measure the wrong row); the same loop with the
/// copy only is timed too and subtracted.
fn time_sort(pristine: &[i64], mut sort: impl FnMut(&mut [i64])) -> f64 {
    let mut buf = vec![0; pristine.len()];
    let mut measure = |reps: u32, with_sort: bool| {
        let start = Instant::now();
        for _ in 0..reps {
            buf.copy_from_slice(black_box(pristine));
            if with_sort {
                sort(&mut buf);
            }
            black_box(&mut buf);
        }
        start.elapsed()
    };
    let mut reps = 1; // grow until one batch takes >= 5 ms
    while measure(reps, true) < Duration::from_millis(5) {
        reps *= 2;
    }
    let best = (0..3)
        .map(|_| measure(reps, true).saturating_sub(measure(reps, false)))
        .min()
        .unwrap_or_default();
    best.as_nanos() as f64 / f64::from(reps) / 1000.0
}

fn main() {
    let inp = inputs();
    println!(
        "n = {N}; n-1 = {}, n*log2(n) = {:.0}; * = O(n) here, i.e. <= 3n = {} comparisons\n",
        N - 1,
        N as f64 * (N as f64).log2(),
        3 * N
    );
    println!(
        "{:<26} {:>10} {:>11} {:>15} {:>8}",
        "input", "I", "insertion", "sort_unstable", "sort"
    );
    println!("{}", "-".repeat(73));
    let names = ["insertion", "sort_unstable", "sort"];
    let mut not_linear: [Vec<&str>; 3] = Default::default();
    let mut unstable_count = Vec::new(); // for the 4-vs-5 line
    for (name, data) in &inp {
        let (ic, shifts) = insertion(&mut data.clone());
        let uc = count_unstable(data.clone());
        let sc = count_stable(data.clone());
        println!(
            "{name:<26} {shifts:>10} {ic:>10}{} {uc:>14}{} {sc:>7}{}",
            linear(ic),
            linear(uc),
            linear(sc)
        );
        for (col, c) in [ic, uc, sc].into_iter().enumerate() {
            if c > 3 * N {
                not_linear[col].push(name);
            }
        }
        unstable_count.push((name.as_str(), uc));
    }
    println!();
    for (name, rows) in names.iter().zip(&not_linear) {
        println!(
            "{name:<13} O(n) on {} of {} inputs; not on: {}",
            inp.len() - rows.len(),
            inp.len(),
            rows.join(", ")
        );
    }
    let get = |row: &str| {
        unstable_count
            .iter()
            .find(|(n, _)| *n == row)
            .map_or(0, |&(_, c)| c)
    };
    let (a, b) = (get("4 adjacent swaps"), get("5 adjacent swaps"));
    let ratio = b as f64 / a as f64;
    // Go's partialInsertionSort has a 4-vs-5 cliff; say what THIS run shows.
    let verdict = if ratio >= 1.5 { "a" } else { "no" };
    println!(
        "\n{verdict} 4-vs-5 cliff here: sort_unstable needs {a} comparisons for 4 adjacent swaps, {b} for 5 (x{ratio:.1})"
    );

    // One timing table, as a supplement to the counts. sort_unstable / sort
    // on i64 use Ord directly, as does our insertion sort.
    println!();
    println!("time per sort, µs (best of 3, copy cost subtracted) -- varies by machine");
    println!(
        "{:<26} {:>10} {:>14} {:>8}",
        "input", "insertion", "sort_unstable", "sort"
    );
    for (name, data) in &inp {
        let ti = time_sort(data, |s| {
            insertion(s);
        });
        let tu = time_sort(data, |s| s.sort_unstable());
        let ts = time_sort(data, |s| s.sort());
        println!("{name:<26} {ti:>10.1} {tu:>14.1} {ts:>8.1}");
    }
}
```

**Rust sample output:**

```
n = 10000; n-1 = 9999, n*log2(n) = 132877; * = O(n) here, i.e. <= 3n = 30000 comparisons

input                               I   insertion   sort_unstable     sort
-------------------------------------------------------------------------
ascending                           0       9999*           9999*    9999*
strictly descending          49995000   49995000            9999*    9999*
non-strictly descending      49990000   49995000          146993   150655 
last moved to front              9999      19997*         133261    20449*
first moved to end               9999      19997*         144523    19998*
4 adjacent swaps                    4      10003*         137047    18097*
5 adjacent swaps                    5      10004*         134512    19308*
sorted + 100 random            549209     559208          151916    20540*
all equal                           0       9999*           9999*    9999*
organ pipe                   24995000   25004999          163479    19998*

insertion     O(n) on 6 of 10 inputs; not on: strictly descending, non-strictly descending, sorted + 100 random, organ pipe
sort_unstable O(n) on 3 of 10 inputs; not on: non-strictly descending, last moved to front, first moved to end, 4 adjacent swaps, 5 adjacent swaps, sorted + 100 random, organ pipe
sort          O(n) on 9 of 10 inputs; not on: non-strictly descending

no 4-vs-5 cliff here: sort_unstable needs 137047 comparisons for 4 adjacent swaps, 134512 for 5 (x1.0)

time per sort, µs (best of 3, copy cost subtracted) -- varies by machine
input                       insertion  sort_unstable     sort
ascending                         5.4            2.9      3.0
strictly descending           14304.6            3.3      3.6
non-strictly descending       14190.9           58.2     60.9
last moved to front               8.0           51.2     16.4
first moved to end                8.4           54.4      8.0
4 adjacent swaps                  5.4           51.9     15.8
5 adjacent swaps                  5.4           51.3     16.1
sorted + 100 random             164.1           60.3     24.1
all equal                         5.5            2.9      3.1
organ pipe                     7185.4           62.9     30.7
```

> **Go vs Rust:** Same inputs (PCG port) and identical I and insertion columns; the std columns differ from Go's by design. Both Rust sorts open with `find_existing_run` (`core/src/slice/sort/shared/mod.rs`): exactly n−1 = 9,999 on ascending, strictly descending and all-equal input (Go's `SortFunc` 10,011), but a descending input whose first pair is **equal** starts an *ascending* run, so the non-strictly descending row costs ~147k–151k (Go 37,573). `sort_unstable` (ipnsort) has nothing after that scan — no `partialInsertionSort` — so every near miss costs about n·log2(n) and there is no 4-vs-5 cliff, while `sort` (driftsort, `stable/drift.rs`) keeps every run of at least ~√n and merges them: O(n) on 9 of 10 rows, including the random tail (20,540 vs Go stable's 13,862) and organ pipe (19,998).
> Timings over 5 runs on a shared machine: 4–5 adjacent swaps — insertion ~5–8 µs, `sort` ~13–17 µs, `sort_unstable` ~50–70 µs; random tail ~24 µs `sort` vs ~160–170 µs insertion; reversed ~14–15 ms insertion vs ~3.5 µs for either std sort.

**Complexity:** insertion sort is Θ(n + I) on every row (comparisons = I plus at most n−1) · `slices.SortFunc` is Θ(n) on ascending, strictly descending and all-equal input (10,011 comparisons) and up to 4 misplaced adjacent pairs, but a 5th pair costs 2.9× more (28,941: `partialInsertionSort` fixes it in its last round and never verifies) and the random tail costs 292,634 · `slices.SortStableFunc` stays under 1.4n on all five near misses of ascending input (13,862 on the random tail) yet spends 98,192 on reversed input · timings over 3 runs: 5 adjacent swaps ~5.6 µs insertion vs ~14 µs `slices.Sort`; random tail ~44–47 µs stable vs ~146 µs insertion vs ~175 µs `slices.Sort`; reversed input ~12.8 ms for insertion vs ~7 µs for `slices.Sort`.

---

## 14. Shell sort: insertion sort with a gap

`🔴 hard` · *Breaking the adjacent-swap barrier*

Example 6 proved that a sort which only swaps neighbours removes one inversion per swap, so it needs about
n²/4 moves on random input. Shell sort keeps insertion sort's inner loop but runs it on elements h apart, for a
shrinking sequence of gaps that ends with h = 1 — and that last pass is plain insertion sort, so the result is
sorted; the wide passes before it move elements far in one step and leave the final pass almost nothing to do.
Counting shows the barrier fall: each move removes about 700 to 1,200 inversions at n = 10⁵, and comparisons stay within a
small factor of n log₂ n — although that factor grows with n (Knuth's gaps are O(n^{3/2}) in the worst case, Shell's
original halving gaps Θ(n²)). The price is stability: a gapped move jumps straight over equal keys.

**Steps:**

1. Write one gapped insertion sort (hold, shift h places, drop) with comparison and move counters, and three gap
   sequences: Shell's n/2, n/4, …; Knuth's 3h+1; Ciura's measured gaps extended by ×2.25.
2. Count comparisons and moves on seeded permutations of n = 10³, 10⁴, 10⁵ next to I and n log₂ n, then sort tagged
   records with few distinct keys to see stability break.
3. Time the three variants against `slices.Sort` at n = 10⁵, restoring the pristine input before every run.

```go
package main

import (
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"strings"
	"time"
)

// Shell sort is insertion sort run on elements h apart, for a shrinking
// sequence of gaps that ends in 1. The final h = 1 pass IS plain insertion
// sort, so the result is sorted; the wide passes before it only make that last
// pass cheap. A move by h places can remove many inversions at once -- which
// example 6 proved an adjacent swap cannot -- so Shell sort is not bound by I.
//
// Cost model: a comparison is one "s[j-h] <= x" test; a move is one slot
// write -- a shift step is 1 move, dropping the held key is 1 move (only
// when it actually travelled).

type stats struct{ comps, moves int }

// shell sorts s with the given gaps (largest first) and counts.
func shell(s []int, gaps []int) stats {
	var st stats
	for _, h := range gaps {
		for i := h; i < len(s); i++ {
			x := s[i] // hold it, shift larger elements h places right
			j := i
			for j >= h {
				st.comps++
				if s[j-h] <= x {
					break // x has found its place in this h-chain
				}
				s[j] = s[j-h]
				st.moves++
				j -= h
			}
			if j != i {
				s[j] = x
				st.moves++
			}
		}
	}
	return st
}

// Three gap sequences, each returned largest first.

// Shell's 1959 original: n/2, n/4, ..., 1. Θ(n²) in the worst case.
func shellGaps(n int) []int {
	var g []int
	for h := n / 2; h > 0; h /= 2 {
		g = append(g, h)
	}
	return g
}

// Knuth's 1, 4, 13, 40, ... (h = 3h+1), gaps below n/3. O(n^{3/2}) worst case.
func knuthGaps(n int) []int {
	g := []int{1}
	for h := 4; h < n/3; h = 3*h + 1 {
		g = append(g, h)
	}
	slices.Reverse(g)
	return g
}

// Ciura's empirically found 1, 4, 10, 23, 57, 132, 301, 701; beyond 701 the
// usual EXTENSION h = floor(2.25 h) (not part of Ciura's measured sequence).
func ciuraGaps(n int) []int {
	base := []int{1, 4, 10, 23, 57, 132, 301, 701}
	var g []int
	for _, h := range base {
		if h < n {
			g = append(g, h)
		}
	}
	if len(g) == len(base) {
		for h := 701 * 9 / 4; h < n; h = h * 9 / 4 { // h*9/4 = floor(2.25 h)
			g = append(g, h)
		}
	}
	slices.Reverse(g)
	return g
}

type variant struct {
	name string
	gaps func(n int) []int
}

var variants = []variant{
	{"insertion (gap 1)", func(int) []int { return []int{1} }},
	{"Shell n/2, n/4, ...", shellGaps},
	{"Knuth 3h+1", knuthGaps},
	{"Ciura (+ x2.25)", ciuraGaps},
}

// perm is Fisher-Yates on 0..n-1 driven by r.IntN (the Rust twin replays it).
func perm(r *rand.Rand, n int) []int {
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

// inversions of a permutation of 0..n-1 in O(n log n) with a Fenwick tree:
// walk right to left, count the smaller values already seen.
func inversions(p []int) int {
	tree := make([]int, len(p)+1)
	inv := 0
	for i := len(p) - 1; i >= 0; i-- {
		for k := p[i]; k > 0; k -= k & -k { // values < p[i]
			inv += tree[k]
		}
		for k := p[i] + 1; k <= len(p); k += k & -k {
			tree[k]++
		}
	}
	return inv
}

func gapString(g []int) string {
	parts := make([]string, len(g))
	for i, h := range g {
		parts[i] = fmt.Sprint(h)
	}
	return strings.Join(parts, " ")
}

// timeSort: best of 3, in ms; the pristine input is copied back before EVERY
// sort (sorting sorted data measures nothing) and a copy-only loop is subtracted.
func timeSort(pristine []int, sort func([]int)) float64 {
	buf := make([]int, len(pristine))
	measure := func(reps int, withSort bool) time.Duration {
		start := time.Now()
		for range reps {
			copy(buf, pristine)
			if withSort {
				sort(buf)
			}
		}
		return time.Since(start)
	}
	reps := 1
	for measure(reps, true) < 20*time.Millisecond {
		reps *= 2
	}
	best := time.Duration(1 << 62)
	for range 3 {
		best = min(best, measure(reps, true)-measure(reps, false))
	}
	return float64(best.Nanoseconds()) / float64(reps) / 1e6
}

func main() {
	r := rand.New(rand.NewPCG(7, 8))
	fmt.Println("gap sequences at n = 100000 (largest first):")
	for _, v := range variants[1:] {
		fmt.Printf("  %-20s %s\n", v.name, gapString(v.gaps(100_000)))
	}

	fmt.Println()
	fmt.Println("Table 1: random permutations; c/nlogn = comparisons / (n log2 n);")
	fmt.Println("         I/move = inversions removed per move (an adjacent swap removes exactly 1)")
	fmt.Printf("%-8s %-20s %13s %13s %8s %7s\n", "n", "sort", "comparisons", "moves", "c/nlogn", "I/move")
	var big []int // the n = 100000 permutation, kept for the timing table
	for _, n := range []int{1_000, 10_000, 100_000} {
		p := perm(r, n)
		nlogn := float64(n) * math.Log2(float64(n))
		inv := inversions(p)
		fmt.Printf("%-8d I = %d inversions, n log2 n = %.0f\n", n, inv, nlogn)
		for _, v := range variants {
			if v.name == variants[0].name && n > 10_000 { // ~n^2/4 = 2.5e9 steps: too slow
				fmt.Printf("%-8s %-20s %13s\n", "", v.name, "skipped: ~I steps")
				continue
			}
			s := slices.Clone(p)
			st := shell(s, v.gaps(n))
			if !slices.IsSorted(s) {
				panic("not sorted: " + v.name)
			}
			fmt.Printf("%-8s %-20s %13d %13d %8.2f %7.1f\n", "", v.name, st.comps, st.moves,
				float64(st.comps)/nlogn, float64(inv)/float64(st.moves))
		}
		big = p
	}

	// Table 2: stability. A gapped pass moves an element h places, straight
	// over any equal keys that sit between -- so equal keys can swap order.
	fmt.Println()
	const trials, m = 1000, 50
	fmt.Printf("Table 2: stability, %d random inputs of %d records, keys in [0, 4)\n", trials, m)
	fmt.Printf("%-20s %18s   %s\n", "sort", "trials reordered", "first violation")
	type rec struct{ key, tag int }
	inputs := make([][]rec, trials)
	for t := range inputs {
		in := make([]rec, m)
		for i := range in {
			in[i] = rec{r.IntN(4), i} // tag = input position
		}
		inputs[t] = in
	}
	for _, v := range variants {
		bad, first := 0, ""
		for t, in := range inputs {
			// sort the records by key with the same gapped insertion sort
			s := slices.Clone(in)
			for _, h := range v.gaps(m) {
				for i := h; i < m; i++ {
					x, j := s[i], i
					for ; j >= h && s[j-h].key > x.key; j -= h {
						s[j] = s[j-h]
					}
					s[j] = x
				}
			}
			for i := 1; i < m; i++ {
				if s[i].key == s[i-1].key && s[i].tag < s[i-1].tag {
					if bad == 0 {
						first = fmt.Sprintf("trial %d: key %d, tag %d now after tag %d",
							t+1, s[i].key, s[i].tag, s[i-1].tag)
					}
					bad++
					break
				}
			}
		}
		if first == "" {
			first = "none"
		}
		fmt.Printf("%-20s %18d   %s\n", v.name, bad, first)
	}

	// One timing table (varies by machine): n = 100000, the same permutation.
	fmt.Println()
	fmt.Println("time per sort at n = 100000, ms (best of 3, copy cost subtracted):")
	base := timeSort(big, slices.Sort[[]int])
	for _, v := range variants[1:] {
		g := v.gaps(len(big))
		t := timeSort(big, func(s []int) { shell(s, g) })
		fmt.Printf("  %-20s %8.2f   (%.1fx slices.Sort)\n", v.name, t, t/base)
	}
	fmt.Printf("  %-20s %8.2f\n", "slices.Sort", base)
}
```

**Sample output:**

```
gap sequences at n = 100000 (largest first):
  Shell n/2, n/4, ...  50000 25000 12500 6250 3125 1562 781 390 195 97 48 24 12 6 3 1
  Knuth 3h+1           29524 9841 3280 1093 364 121 40 13 4 1
  Ciura (+ x2.25)      90927 40412 17961 7983 3548 1577 701 301 132 57 23 10 4 1

Table 1: random permutations; c/nlogn = comparisons / (n log2 n);
         I/move = inversions removed per move (an adjacent swap removes exactly 1)
n        sort                   comparisons         moves  c/nlogn  I/move
1000     I = 249337 inversions, n log2 n = 9966
         insertion (gap 1)           250327        250332    25.12     1.0
         Shell n/2, n/4, ...          15642         12179     1.57    20.5
         Knuth 3h+1                   13953         12687     1.40    19.7
         Ciura (+ x2.25)              13204         10438     1.32    23.9
10000    I = 25162530 inversions, n log2 n = 132877
         insertion (gap 1)         25172517      25172521   189.44     1.0
         Shell n/2, n/4, ...         266717        210587     2.01   119.5
         Knuth 3h+1                  225136        203755     1.69   123.5
         Ciura (+ x2.25)             192295        152343     1.45   165.2
100000   I = 2497820071 inversions, n log2 n = 1660964
         insertion (gap 1)    skipped: ~I steps
         Shell n/2, n/4, ...        4305285       3630730     2.59   688.0
         Knuth 3h+1                 3707553       3455384     2.23   722.9
         Ciura (+ x2.25)            2553089       2037604     1.54  1225.9

Table 2: stability, 1000 random inputs of 50 records, keys in [0, 4)
sort                   trials reordered   first violation
insertion (gap 1)                     0   none
Shell n/2, n/4, ...                1000   trial 1: key 0, tag 27 now after tag 37
Knuth 3h+1                         1000   trial 1: key 0, tag 14 now after tag 17
Ciura (+ x2.25)                    1000   trial 1: key 0, tag 24 now after tag 33

time per sort at n = 100000, ms (best of 3, copy cost subtracted):
  Shell n/2, n/4, ...      7.89   (1.7x slices.Sort)
  Knuth 3h+1               6.63   (1.4x slices.Sort)
  Ciura (+ x2.25)          7.41   (1.6x slices.Sort)
  slices.Sort              4.61
```

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// Shell sort is insertion sort run on elements h apart, for a shrinking
// sequence of gaps that ends in 1. The final h = 1 pass IS plain insertion
// sort, so the result is sorted; the wide passes before it only make that last
// pass cheap. A move by h places can remove many inversions at once -- which
// example 6 proved an adjacent swap cannot -- so Shell sort is not bound by I.
//
// Cost model: a comparison is one "s[j-h] <= x" test; a move is one slot
// write -- a shift step is 1 move, dropping the held key is 1 move (only
// when it actually travelled).

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and IntN, so this program
// sorts exactly the permutations and records the Go program sorts.
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

#[derive(Default, Clone, Copy)]
struct Stats {
    comps: u64,
    moves: u64,
}

/// Gapped insertion sort over any `Copy` element, ordered by `key`; the same
/// function sorts the plain permutations and the tagged records below.
/// `(h..=i).rev().step_by(h)` walks i, i-h, i-2h, ... while >= h: Go's
/// `for j >= h { ...; j -= h }` as an iterator.
fn shell<T: Copy, K: PartialOrd>(s: &mut [T], gaps: &[usize], key: impl Fn(&T) -> K) -> Stats {
    let mut st = Stats::default();
    for &h in gaps {
        for i in h..s.len() {
            let x = s[i]; // hold it, shift larger elements h places right
            let kx = key(&x);
            let mut hole = i;
            for j in (h..=i).rev().step_by(h) {
                st.comps += 1;
                if key(&s[j - h]) <= kx {
                    break; // x has found its place in this h-chain
                }
                s[j] = s[j - h];
                st.moves += 1;
                hole = j - h;
            }
            if hole != i {
                s[hole] = x;
                st.moves += 1;
            }
        }
    }
    st
}

// Three gap sequences, each returned largest first.

/// Shell's 1959 original: n/2, n/4, ..., 1. Θ(n²) in the worst case.
fn shell_gaps(n: usize) -> Vec<usize> {
    std::iter::successors(Some(n / 2), |&h| Some(h / 2))
        .take_while(|&h| h > 0)
        .collect()
}

/// Knuth's 1, 4, 13, 40, ... (h = 3h+1), gaps below n/3. O(n^{3/2}) worst case.
fn knuth_gaps(n: usize) -> Vec<usize> {
    let mut g = vec![1];
    g.extend(std::iter::successors(Some(4), |&h| Some(3 * h + 1)).take_while(|&h| h < n / 3));
    g.reverse();
    g
}

/// Ciura's empirically found 1, 4, 10, 23, 57, 132, 301, 701; beyond 701 the
/// usual EXTENSION h = floor(2.25 h) (not part of Ciura's measured sequence).
fn ciura_gaps(n: usize) -> Vec<usize> {
    const BASE: [usize; 8] = [1, 4, 10, 23, 57, 132, 301, 701];
    let mut g: Vec<usize> = BASE.iter().copied().filter(|&h| h < n).collect();
    if g.len() == BASE.len() {
        // h*9/4 = floor(2.25 h)
        g.extend(
            std::iter::successors(Some(701 * 9 / 4), |&h| Some(h * 9 / 4)).take_while(|&h| h < n),
        );
    }
    g.reverse();
    g
}

fn insertion_gaps(_: usize) -> Vec<usize> {
    vec![1]
}

// fn pointers, not closures: all four have the same type, so they fit one array.
type GapFn = fn(usize) -> Vec<usize>;
const VARIANTS: [(&str, GapFn); 4] = [
    ("insertion (gap 1)", insertion_gaps),
    ("Shell n/2, n/4, ...", shell_gaps),
    ("Knuth 3h+1", knuth_gaps),
    ("Ciura (+ x2.25)", ciura_gaps),
];

/// Fisher-Yates on 0..n-1 driven by int_n -- the same draws as the Go program.
fn perm(r: &mut Pcg, n: usize) -> Vec<u64> {
    let mut p: Vec<u64> = (0..n as u64).collect();
    for i in (1..n).rev() {
        let j = r.int_n(i as u64 + 1) as usize;
        p.swap(i, j);
    }
    p
}

/// Inversions of a permutation of 0..n-1 in O(n log n) with a Fenwick tree:
/// walk right to left, count the smaller values already seen.
fn inversions(p: &[u64]) -> u64 {
    let n = p.len();
    let mut tree = vec![0u64; n + 1];
    let mut inv = 0;
    for &v in p.iter().rev() {
        let mut k = v as usize; // values < v
        while k > 0 {
            inv += tree[k];
            k &= k - 1; // k -= k & -k
        }
        let mut k = v as usize + 1;
        while k <= n {
            tree[k] += 1;
            k += k.isolate_lowest_one(); // k += k & -k
        }
    }
    inv
}

fn gap_string(g: &[usize]) -> String {
    g.iter().map(usize::to_string).collect::<Vec<_>>().join(" ")
}

/// Best of 3, in ms; the pristine input is copied back before EVERY sort
/// (sorting sorted data measures nothing) and a copy-only loop is subtracted.
fn time_sort(pristine: &[u64], mut sort: impl FnMut(&mut [u64])) -> f64 {
    let mut buf = vec![0; pristine.len()];
    let mut measure = |reps: u32, with_sort: bool| {
        let start = Instant::now();
        for _ in 0..reps {
            buf.copy_from_slice(black_box(pristine));
            if with_sort {
                sort(&mut buf);
            }
            black_box(&mut buf);
        }
        start.elapsed()
    };
    let mut reps = 1;
    while measure(reps, true) < Duration::from_millis(20) {
        reps *= 2;
    }
    let best = (0..3)
        .map(|_| measure(reps, true).saturating_sub(measure(reps, false)))
        .min()
        .unwrap_or_default();
    best.as_nanos() as f64 / f64::from(reps) / 1e6
}

fn main() {
    let mut r = Pcg::new(7, 8);
    println!("gap sequences at n = 100000 (largest first):");
    for (name, gaps) in &VARIANTS[1..] {
        println!("  {name:<20} {}", gap_string(&gaps(100_000)));
    }

    println!();
    println!("Table 1: random permutations; c/nlogn = comparisons / (n log2 n);");
    println!("         I/move = inversions removed per move (an adjacent swap removes exactly 1)");
    println!(
        "{:<8} {:<20} {:>13} {:>13} {:>8} {:>7}",
        "n", "sort", "comparisons", "moves", "c/nlogn", "I/move"
    );
    let mut big = Vec::new(); // the n = 100000 permutation, kept for the timing table
    for n in [1_000, 10_000, 100_000] {
        let p = perm(&mut r, n);
        let nlogn = n as f64 * (n as f64).log2();
        let inv = inversions(&p);
        println!("{n:<8} I = {inv} inversions, n log2 n = {nlogn:.0}");
        for (k, (name, gaps)) in VARIANTS.iter().enumerate() {
            if k == 0 && n > 10_000 {
                // ~n^2/4 = 2.5e9 steps: too slow
                println!("{:<8} {name:<20} {:>13}", "", "skipped: ~I steps");
                continue;
            }
            let mut s = p.clone();
            let st = shell(&mut s, &gaps(n), |&v| v);
            assert!(s.is_sorted(), "not sorted: {name}");
            println!(
                "{:<8} {name:<20} {:>13} {:>13} {:>8.2} {:>7.1}",
                "",
                st.comps,
                st.moves,
                st.comps as f64 / nlogn,
                inv as f64 / st.moves as f64
            );
        }
        big = p;
    }

    // Table 2: stability. A gapped pass moves an element h places, straight
    // over any equal keys that sit between -- so equal keys can swap order.
    println!();
    const TRIALS: usize = 1000;
    const M: usize = 50;
    println!("Table 2: stability, {TRIALS} random inputs of {M} records, keys in [0, 4)");
    println!(
        "{:<20} {:>18}   first violation",
        "sort", "trials reordered"
    );
    // (key, tag) with tag = input position
    let inputs: Vec<Vec<(u8, u32)>> = (0..TRIALS)
        .map(|_| (0..M as u32).map(|i| (r.int_n(4) as u8, i)).collect())
        .collect();
    for (name, gaps) in &VARIANTS {
        let g = gaps(M);
        let mut bad = 0;
        let mut first = None;
        for (t, input) in inputs.iter().enumerate() {
            let mut s = input.clone();
            shell(&mut s, &g, |rec| rec.0); // the same sort, ordered by key only
            if let Some(w) = s.windows(2).find(|w| w[0].0 == w[1].0 && w[1].1 < w[0].1) {
                bad += 1;
                first.get_or_insert_with(|| {
                    format!(
                        "trial {}: key {}, tag {} now after tag {}",
                        t + 1,
                        w[1].0,
                        w[1].1,
                        w[0].1
                    )
                });
            }
        }
        println!(
            "{name:<20} {bad:>18}   {}",
            first.as_deref().unwrap_or("none")
        );
    }

    // One timing table (varies by machine): n = 100000, the same permutation.
    println!();
    println!("time per sort at n = 100000, ms (best of 3, copy cost subtracted):");
    let base = time_sort(&big, |s| s.sort_unstable());
    for (name, gaps) in &VARIANTS[1..] {
        let g = gaps(big.len());
        let t = time_sort(&big, |s| {
            shell(s, &g, |&v| v);
        });
        println!("  {name:<20} {t:>8.2}   ({:.1}x sort_unstable)", t / base);
    }
    println!("  {:<20} {base:>8.2}", "sort_unstable");
}
```

**Rust sample output:**

```
gap sequences at n = 100000 (largest first):
  Shell n/2, n/4, ...  50000 25000 12500 6250 3125 1562 781 390 195 97 48 24 12 6 3 1
  Knuth 3h+1           29524 9841 3280 1093 364 121 40 13 4 1
  Ciura (+ x2.25)      90927 40412 17961 7983 3548 1577 701 301 132 57 23 10 4 1

Table 1: random permutations; c/nlogn = comparisons / (n log2 n);
         I/move = inversions removed per move (an adjacent swap removes exactly 1)
n        sort                   comparisons         moves  c/nlogn  I/move
1000     I = 249337 inversions, n log2 n = 9966
         insertion (gap 1)           250327        250332    25.12     1.0
         Shell n/2, n/4, ...          15642         12179     1.57    20.5
         Knuth 3h+1                   13953         12687     1.40    19.7
         Ciura (+ x2.25)              13204         10438     1.32    23.9
10000    I = 25162530 inversions, n log2 n = 132877
         insertion (gap 1)         25172517      25172521   189.44     1.0
         Shell n/2, n/4, ...         266717        210587     2.01   119.5
         Knuth 3h+1                  225136        203755     1.69   123.5
         Ciura (+ x2.25)             192295        152343     1.45   165.2
100000   I = 2497820071 inversions, n log2 n = 1660964
         insertion (gap 1)    skipped: ~I steps
         Shell n/2, n/4, ...        4305285       3630730     2.59   688.0
         Knuth 3h+1                 3707553       3455384     2.23   722.9
         Ciura (+ x2.25)            2553089       2037604     1.54  1225.9

Table 2: stability, 1000 random inputs of 50 records, keys in [0, 4)
sort                   trials reordered   first violation
insertion (gap 1)                     0   none
Shell n/2, n/4, ...                1000   trial 1: key 0, tag 27 now after tag 37
Knuth 3h+1                         1000   trial 1: key 0, tag 14 now after tag 17
Ciura (+ x2.25)                    1000   trial 1: key 0, tag 24 now after tag 33

time per sort at n = 100000, ms (best of 3, copy cost subtracted):
  Shell n/2, n/4, ...      9.16   (12.4x sort_unstable)
  Knuth 3h+1               7.71   (10.4x sort_unstable)
  Ciura (+ x2.25)          8.00   (10.8x sort_unstable)
  sort_unstable            0.74
```

> **Go vs Rust:** Everything above the timing table is identical to Go's, byte for byte (same PCG permutations and records, same counts, same first violations). One generic `shell<T: Copy, K: PartialOrd>` with a key closure sorts both the `u64` permutations and the `(key, tag)` records, and `(h..=i).rev().step_by(h)` is Go's `for j >= h { …; j -= h }`.
> The timing table tells a different story from Go's, and the difference is mostly the std sort: `sort_unstable` (ipnsort) took ~0.7–0.8 ms on the 10⁵ `u64`s against ~4.7 ms for Go's `slices.Sort` on `int`s, while the Rust Shell sorts took ~8–13 ms over 5 runs on a shared machine (Go 6.6–7.9 ms) — so here Shell sort is an order of magnitude slower than the std sort (10–15×), not 1.4–1.7×. Halving gaps were the slowest in every run; Knuth vs Ciura swapped places between runs, so no ranking is claimed for them.

**Complexity:** Knuth's gaps O(n^{3/2}) worst case · Shell's halving gaps Θ(n²) worst case · in place, Θ(1) extra space, **not stable** (every gap sequence reordered ties in 1000 of 1000 trials; plain insertion in 0) · measured on random permutations: comparisons / (n log₂ n) grows with n for all three sequences (Knuth 1.40 → 1.69 → 2.23, Ciura 1.32 → 1.45 → 1.54 from 10³ to 10⁵), and at n = 10⁵ each move removes ~690 (Shell) to ~1,230 (Ciura) inversions where an adjacent swap removes one · in time, all three were 1.4–1.7× slower than `slices.Sort` at n = 10⁵ over 3 runs, Knuth's the fastest of them although Ciura's makes fewer comparisons — the O(n log n) sorts are lesson 13.

---

## 15. When writes cost more than comparisons

`🔴 hard` · *Selection sort's one real advantage*

Selection sort makes n(n−1)/2 comparisons on every input but at most n−1 swaps. Insertion sort makes about n²/4
comparisons and about n²/4 moves. With 8-byte elements a move costs about what a comparison costs, and insertion sort
wins. With a 1 KiB record a move is a 1 KiB memcpy, so the right question is how many **bytes** each sort moves. This
example sorts n = 2000 records of 8, 64, 256 and 1024 bytes. It counts comparisons, writes and bytes moved, which are
deterministic, and then times the sorts. It also shows the usual way out of the trade-off: sort an index slice, which
keeps insertion sort's comparisons and makes its Θ(n²) moves 8-byte moves, then permute the records once along their
cycles, which is at most n record writes. Selection and insertion are both Θ(n²) comparisons. Selection makes Θ(n)
moves, insertion Θ(n²), and the index sort Θ(n²) index moves plus Θ(n) record moves.

**Steps:**

1. Build a generic `Rec[P]` with the payload type as the parameter, and write selection, insertion, and index sort
   plus permute, each returning its counts.
2. Check that every count is the same at every record size and that bytes moved = writes × size. Insertion's
   shifts equal I exactly.
3. Time all of them plus `slices.SortFunc` on the records and on an index slice. Find the record size at which
   selection sort overtakes insertion sort.

```go
package main

import (
	"cmp"
	"fmt"
	"math/rand/v2"
	"slices"
	"time"
	"unsafe"
)

// Selection sort makes n(n-1)/2 comparisons on every input but at most n-1
// swaps. Insertion sort makes about n^2/4 comparisons AND n^2/4 moves on random
// input. When an element is an 8-byte int, a move costs about what a comparison
// costs and insertion wins. When an element is a 1 KiB record, a move is a
// 1 KiB memcpy -- and the question becomes: how many BYTES does each sort move?
//
// A "write" is one slot assignment: a swap = 2 writes, a shift step = 1, dropping
// the held element into its hole = 1. The temporaries (the swap's tmp, the
// insertion sort's held x) are not counted. bytes moved = writes * element size.

// Rec is a record with an int key and P bytes of payload the sort never reads.
// Go generics cannot take an array LENGTH as a parameter, so the payload type
// itself is the parameter: Rec[[56]byte] is a 64-byte record. The payload comes
// FIRST because Go pads a zero-size field at the END of a struct (so that its
// address cannot point past the object): with Key first, Rec[[0]byte] is 16 bytes.
type Rec[P any] struct {
	Payload P
	Key     int
}

type counts struct{ comps, shifts, recWrites, idxWrites int }

// selectionSort: the prefix a[:i] holds the i smallest records, in their final
// places. Find the minimum of the rest, swap it to position i.
func selectionSort[P any](a []Rec[P]) counts {
	var c counts
	for i := 0; i < len(a)-1; i++ {
		m, mk := i, a[i].Key // keep the minimum's key in a local too
		for j := i + 1; j < len(a); j++ {
			c.comps++
			if a[j].Key < mk {
				m, mk = j, a[j].Key
			}
		}
		if m != i { // no self-swap: a swap is 3 record copies, skip it if we can
			a[i], a[m] = a[m], a[i]
			c.recWrites += 2
		}
	}
	return c
}

// insertionSort: the shift version from example 3, on whole records.
func insertionSort[P any](a []Rec[P]) counts {
	var c counts
	for i := 1; i < len(a); i++ {
		x := a[i] // hold the record (one memcpy into a temporary)
		j := i
		for j > 0 {
			c.comps++
			if !(x.Key < a[j-1].Key) {
				break
			}
			a[j] = a[j-1] // every shift step moves a WHOLE record
			c.recWrites++
			c.shifts++
			j--
		}
		if j != i {
			a[j] = x
			c.recWrites++
		}
	}
	return c
}

// indexSort is the standard way out of the dilemma: sort 8-byte indices with
// insertion sort (cheap moves, the same comparisons), then move each record
// ONCE. idx is caller-owned scratch of length len(a), so nothing is allocated.
func indexSort[P any](a []Rec[P], idx []int) counts {
	var c counts
	for i := range idx {
		idx[i] = i
		c.idxWrites++
	}
	for i := 1; i < len(idx); i++ {
		x := idx[i]
		k := a[x].Key
		j := i
		for j > 0 {
			c.comps++
			if !(k < a[idx[j-1]].Key) { // compare the keys the indices point at
				break
			}
			idx[j] = idx[j-1] // an 8-byte move, whatever the record size
			c.idxWrites++
			c.shifts++
			j--
		}
		if j != i {
			idx[j] = x
			c.idxWrites++
		}
	}
	permute(a, idx, &c)
	return c
}

// permute applies a sorted index slice in place: a[j] becomes old a[idx[j]].
// Follow each cycle: hold its first record, pull each successor into the hole,
// drop the held record into the last hole. idx[j] = j marks slot j as done.
// At most n record writes in total -- one per record that is not already home.
func permute[P any](a []Rec[P], idx []int, c *counts) {
	for i := range idx {
		if idx[i] == i {
			continue // already home, or finished as part of an earlier cycle
		}
		held := a[i]
		j := i
		for {
			k := idx[j]
			idx[j] = j
			c.idxWrites++
			if k == i {
				a[j] = held
				c.recWrites++
				break
			}
			a[j] = a[k]
			c.recWrites++
			j = k
		}
	}
}

// stdIndexSort: the same idea with the stdlib doing the index sort.
func stdIndexSort[P any](a []Rec[P], idx []int) {
	for i := range idx {
		idx[i] = i
	}
	slices.SortFunc(idx, func(i, j int) int { return cmp.Compare(a[i].Key, a[j].Key) })
	var c counts
	permute(a, idx, &c)
}

// byKey is the comparator slices.SortFunc gets for the records themselves.
// Note its signature: func(a, b E) int takes the two records BY VALUE.
func byKey[P any](a, b Rec[P]) int { return cmp.Compare(a.Key, b.Key) }

// nsPerOp: best of 3 rounds; each round repeats f until 40 ms have passed.
func nsPerOp(f func()) float64 {
	best := 0.0
	for round := 0; round < 3; round++ {
		ops := 0
		start := time.Now()
		for time.Since(start) < 40*time.Millisecond {
			f()
			ops++
		}
		ns := float64(time.Since(start).Nanoseconds()) / float64(ops)
		if round == 0 || ns < best {
			best = ns
		}
	}
	return best
}

const n = 2000

type result struct {
	size          int
	sel, ins, idx counts
	ok            bool
	t             [6]float64 // copy, selection, insertion, index, SortFunc rec, SortFunc idx
}

var columns = [6]string{"copy", "selection", "insertion", "index", "SortFunc", "SortFunc idx"}

// measure runs everything for one record size.
func measure[P any](keys []int) result {
	pristine := make([]Rec[P], n)
	for i, k := range keys {
		pristine[i].Key = k
	}
	work := make([]Rec[P], n)
	idx := make([]int, n)
	r := result{size: int(unsafe.Sizeof(pristine[0]))}

	// The keys are a permutation of 0..n-1, so "sorted" means work[i].Key == i
	// for every i -- a lost or duplicated record would show up here.
	sorted := func() bool {
		for i := range work {
			if work[i].Key != i {
				return false
			}
		}
		return true
	}
	copy(work, pristine)
	r.sel = selectionSort(work)
	r.ok = sorted()
	copy(work, pristine)
	r.ins = insertionSort(work)
	r.ok = r.ok && sorted()
	copy(work, pristine)
	r.idx = indexSort(work, idx)
	r.ok = r.ok && sorted()
	copy(work, pristine)
	slices.SortFunc(work, byKey[P])
	r.ok = r.ok && sorted()
	copy(work, pristine)
	stdIndexSort(work, idx)
	r.ok = r.ok && sorted()

	// Timings: a sort MUTATES its input, so every timed iteration first copies
	// the pristine input back; the "copy" column times that copy alone.
	r.t[0] = nsPerOp(func() { copy(work, pristine) })
	r.t[1] = nsPerOp(func() { copy(work, pristine); selectionSort(work) })
	r.t[2] = nsPerOp(func() { copy(work, pristine); insertionSort(work) })
	r.t[3] = nsPerOp(func() { copy(work, pristine); indexSort(work, idx) })
	r.t[4] = nsPerOp(func() { copy(work, pristine); slices.SortFunc(work, byKey[P]) })
	r.t[5] = nsPerOp(func() { copy(work, pristine); stdIndexSort(work, idx) })
	return r
}

func main() {
	// keys: a random permutation of 0..n-1, Fisher-Yates on a fixed PCG seed
	rng := rand.New(rand.NewPCG(15, 16))
	keys := make([]int, n)
	for i := range keys {
		keys[i] = i
	}
	for i := n - 1; i > 0; i-- {
		j := rng.IntN(i + 1)
		keys[i], keys[j] = keys[j], keys[i]
	}
	inv := 0
	for i := range keys {
		for j := i + 1; j < n; j++ {
			if keys[i] > keys[j] {
				inv++
			}
		}
	}
	fmt.Printf("n = %d records, keys = a PCG permutation with I = %d inversions (n(n-1)/4 = %d)\n",
		n, inv, n*(n-1)/4)

	rs := []result{
		measure[[0]byte](keys),
		measure[[56]byte](keys),
		measure[[248]byte](keys),
		measure[[1016]byte](keys),
	}
	same := true
	for _, r := range rs {
		same = same && r.ok && r.sel == rs[0].sel && r.ins == rs[0].ins && r.idx == rs[0].idx
	}
	fmt.Printf("record sizes %d, %d, %d, %d bytes: every sort correct, counts identical at every size: %v\n",
		rs[0].size, rs[1].size, rs[2].size, rs[3].size, same)

	fmt.Println()
	fmt.Println("operation counts (deterministic; a write = one slot assignment, see the top comment)")
	fmt.Println()
	fmt.Printf("  %-16s %11s %10s %10s %9s %9s %9s %9s\n", "", "comparisons", "rec writes",
		"idx writes", "MB @ 8B", "MB @ 64B", "MB @ 256B", "MB @ 1KiB")
	row := func(name string, c counts) {
		fmt.Printf("  %-16s %11d %10d %10d", name, c.comps, c.recWrites, c.idxWrites)
		for _, r := range rs {
			bytes := c.recWrites*r.size + c.idxWrites*8
			fmt.Printf(" %9.3f", float64(bytes)/1e6)
		}
		fmt.Println()
	}
	s, in, ix := rs[0].sel, rs[0].ins, rs[0].idx
	row("selection", s)
	row("insertion", in)
	row("index + permute", ix)
	fmt.Println()
	fmt.Printf("selection: %d comparisons = n(n-1)/2: %v; %d swaps (at most n-1 = %d)\n",
		s.comps, s.comps == n*(n-1)/2, s.recWrites/2, n-1)
	fmt.Printf("insertion: %d record writes = %d shifts + %d drops; shifts == I: %v\n",
		in.recWrites, in.shifts, in.recWrites-in.shifts, in.shifts == inv)
	fmt.Printf("insertion vs selection: %.2fx the comparisons, %.0fx the record writes\n",
		float64(in.comps)/float64(s.comps), float64(in.recWrites)/float64(s.recWrites))
	fmt.Printf("index sort: the same comparisons as insertion: %v; its %d shifts are of 8-byte\n",
		ix.comps == in.comps, ix.shifts)
	fmt.Printf("  indices, and the permutation pass makes %d record writes (n = %d)\n", ix.recWrites, n)

	fmt.Println()
	fmt.Println("time per sort, microseconds (best of 3 rounds; every iteration re-copies the")
	fmt.Println("pristine input; 'copy' is that copy alone and is included in every other column)")
	fmt.Println()
	fmt.Printf("  %-7s", "record")
	for _, name := range columns {
		fmt.Printf(" %10s", name)
	}
	fmt.Printf("  %s\n", "fastest")
	for _, r := range rs {
		fmt.Printf("  %-7s", fmt.Sprintf("%dB", r.size))
		best := 1
		for i, t := range r.t {
			fmt.Printf(" %10.1f", t/1e3)
			if i > 0 && t < r.t[best] {
				best = i
			}
		}
		fmt.Printf("  %s\n", columns[best])
	}

	fmt.Println()
	over := -1
	for i, r := range rs {
		if r.t[1] < r.t[2] {
			over = i
			break
		}
	}
	if over < 0 {
		fmt.Println("insertion beat selection at every record size in this run")
	} else {
		fmt.Printf("selection first beat insertion at %d-byte records in this run\n", rs[over].size)
	}
	for _, r := range rs {
		fmt.Printf("  %4dB: insertion/selection %5.2fx   insertion/index %5.2fx   SortFunc rec/idx %5.2fx\n",
			r.size, r.t[2]/r.t[1], r.t[2]/r.t[3], r.t[4]/r.t[5])
	}
	fmt.Println()
	fmt.Println("the counts say why: the comparisons do not depend on the record size, the bytes")
	fmt.Println("moved do. Sorting indices keeps an algorithm's comparisons and replaces its record")
	fmt.Println("moves with 8-byte moves plus at most n record writes -- for the Theta(n^2) sort")
	fmt.Println("and for slices.SortFunc alike (whose comparator also receives records by value).")
}
```

**Sample output:**

```
n = 2000 records, keys = a PCG permutation with I = 986319 inversions (n(n-1)/4 = 999500)
record sizes 8, 64, 256, 1024 bytes: every sort correct, counts identical at every size: true

operation counts (deterministic; a write = one slot assignment, see the top comment)

                   comparisons rec writes idx writes   MB @ 8B  MB @ 64B MB @ 256B MB @ 1KiB
  selection            1999000       3990          0     0.032     0.255     1.021     4.086
  insertion             988308     988314          0     7.907    63.252   253.008  1012.034
  index + permute       988308       2000     992314     7.955     8.067     8.451     9.987

selection: 1999000 comparisons = n(n-1)/2: true; 1995 swaps (at most n-1 = 1999)
insertion: 988314 record writes = 986319 shifts + 1995 drops; shifts == I: true
insertion vs selection: 0.49x the comparisons, 248x the record writes
index sort: the same comparisons as insertion: true; its 986319 shifts are of 8-byte
  indices, and the permutation pass makes 2000 record writes (n = 2000)

time per sort, microseconds (best of 3 rounds; every iteration re-copies the
pristine input; 'copy' is that copy alone and is included in every other column)

  record        copy  selection  insertion      index   SortFunc SortFunc idx  fastest
  8B             0.2     1106.0      300.7      584.4       47.2       52.0  SortFunc
  64B            1.5     1117.6      643.4      662.4      181.3       61.4  SortFunc idx
  256B           6.5     1223.5     3337.5      844.8      489.7       77.9  SortFunc idx
  1024B         27.6     1330.0    14679.6      935.6     1875.3      136.6  SortFunc idx

selection first beat insertion at 256-byte records in this run
     8B: insertion/selection  0.27x   insertion/index  0.51x   SortFunc rec/idx  0.91x
    64B: insertion/selection  0.58x   insertion/index  0.97x   SortFunc rec/idx  2.95x
   256B: insertion/selection  2.73x   insertion/index  3.95x   SortFunc rec/idx  6.29x
  1024B: insertion/selection 11.04x   insertion/index 15.69x   SortFunc rec/idx 13.72x

the counts say why: the comparisons do not depend on the record size, the bytes
moved do. Sorting indices keeps an algorithm's comparisons and replaces its record
moves with 8-byte moves plus at most n record writes -- for the Theta(n^2) sort
and for slices.SortFunc alike (whose comparator also receives records by value).
```

**Rust:**

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

// Selection sort makes n(n-1)/2 comparisons on every input but at most n-1
// swaps. Insertion sort makes about n^2/4 comparisons AND n^2/4 moves on random
// input. When an element is an 8-byte int, a move costs about what a comparison
// costs and insertion wins. When an element is a 1 KiB record, a move is a
// 1 KiB memcpy -- and the question becomes: how many BYTES does each sort move?
//
// A "write" is one slot assignment: a swap = 2 writes, a shift step = 1, dropping
// the held element into its hole = 1. The temporaries (the swap's tmp, the
// insertion sort's held x) are not counted. bytes moved = writes * element size.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so this
// program shuffles exactly the keys the Go version shuffles.
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

// Rec is a record with a key and P bytes of payload the sort never reads.
// Rust's const generics take the array LENGTH as the parameter (Go has to make
// the payload TYPE the parameter). `Copy` makes every move a plain memcpy, just
// as a struct assignment is in Go. Field order does not matter here: Rust may
// reorder fields, and it does not pad a zero-size field (Rec<0> is 8 bytes
// with either order -- Go pads a trailing one, which is why its Payload is first).
#[derive(Clone, Copy)]
struct Rec<const P: usize> {
    #[expect(dead_code, reason = "the payload exists only to be moved, never read")]
    payload: [u8; P],
    key: i64,
}

#[derive(Clone, Copy, PartialEq, Default)]
struct Counts {
    comps: usize,
    shifts: usize,
    rec_writes: usize,
    idx_writes: usize,
}

// selection_sort: the prefix a[..i] holds the i smallest records, in their
// final places. Find the minimum of the rest, swap it to position i.
fn selection_sort<const P: usize>(a: &mut [Rec<P>]) -> Counts {
    let mut c = Counts::default();
    for i in 0..a.len().saturating_sub(1) {
        // Keep the minimum's key in a local too. Written as `a[j].key < a[m].key`,
        // LLVM turns the `if` into a branch-free select (csel), so each iteration's
        // load of a[m] waits for the previous select: a dependency chain that made
        // this loop ~3x slower in this program. Go's compiler emits a branch there.
        let (mut m, mut mk) = (i, a[i].key);
        for (j, r) in a.iter().enumerate().skip(i + 1) {
            c.comps += 1;
            if r.key < mk {
                (m, mk) = (j, r.key);
            }
        }
        if m != i {
            // no self-swap: a swap is 3 record copies, skip it if we can
            a.swap(i, m);
            c.rec_writes += 2;
        }
    }
    c
}

// insertion_sort: the shift version from example 3, on whole records.
fn insertion_sort<const P: usize>(a: &mut [Rec<P>]) -> Counts {
    let mut c = Counts::default();
    for i in 1..a.len() {
        let x = a[i]; // hold the record (one memcpy into a temporary; Rec is Copy)
        let mut j = i;
        while j > 0 {
            c.comps += 1;
            if x.key >= a[j - 1].key {
                break;
            }
            a[j] = a[j - 1]; // every shift step moves a WHOLE record
            c.rec_writes += 1;
            c.shifts += 1;
            j -= 1;
        }
        if j != i {
            a[j] = x;
            c.rec_writes += 1;
        }
    }
    c
}

// index_sort is the standard way out of the dilemma: sort 8-byte indices with
// insertion sort (cheap moves, the same comparisons), then move each record
// ONCE. idx is caller-owned scratch of length a.len(), so nothing is allocated.
fn index_sort<const P: usize>(a: &mut [Rec<P>], idx: &mut [usize]) -> Counts {
    let mut c = Counts::default();
    for (i, slot) in idx.iter_mut().enumerate() {
        *slot = i;
        c.idx_writes += 1;
    }
    for i in 1..idx.len() {
        let x = idx[i];
        let k = a[x].key;
        let mut j = i;
        while j > 0 {
            c.comps += 1;
            if k >= a[idx[j - 1]].key {
                // compare the keys the indices point at
                break;
            }
            idx[j] = idx[j - 1]; // an 8-byte move, whatever the record size
            c.idx_writes += 1;
            c.shifts += 1;
            j -= 1;
        }
        if j != i {
            idx[j] = x;
            c.idx_writes += 1;
        }
    }
    permute(a, idx, &mut c);
    c
}

// permute applies a sorted index slice in place: a[j] becomes old a[idx[j]].
// Follow each cycle: hold its first record, pull each successor into the hole,
// drop the held record into the last hole. idx[j] = j marks slot j as done.
// At most n record writes in total -- one per record that is not already home.
fn permute<const P: usize>(a: &mut [Rec<P>], idx: &mut [usize], c: &mut Counts) {
    for i in 0..idx.len() {
        if idx[i] == i {
            continue; // already home, or finished as part of an earlier cycle
        }
        let held = a[i];
        let mut j = i;
        loop {
            let k = idx[j];
            idx[j] = j;
            c.idx_writes += 1;
            if k == i {
                a[j] = held;
                c.rec_writes += 1;
                break;
            }
            a[j] = a[k];
            c.rec_writes += 1;
            j = k;
        }
    }
}

// std_index_sort: the same idea with std doing the index sort. sort_unstable_by
// is the counterpart of Go's slices.SortFunc (both unstable, pdqsort-family).
fn std_index_sort<const P: usize>(a: &mut [Rec<P>], idx: &mut [usize]) {
    for (i, slot) in idx.iter_mut().enumerate() {
        *slot = i;
    }
    // the closure borrows `a` immutably while sorting idx; the borrow ends here,
    // before permute takes `a` mutably -- the order the borrow checker insists on
    idx.sort_unstable_by(|&i, &j| a[i].key.cmp(&a[j].key));
    let mut c = Counts::default();
    permute(a, idx, &mut c);
}

/// best of 3 rounds; each round repeats f until 40 ms have passed.
fn ns_per_op(mut f: impl FnMut()) -> f64 {
    let mut best = f64::MAX;
    for _ in 0..3 {
        let mut ops = 0u32;
        let start = Instant::now();
        while start.elapsed() < Duration::from_millis(40) {
            f();
            ops += 1;
        }
        best = best.min(start.elapsed().as_nanos() as f64 / f64::from(ops));
    }
    best
}

const N: usize = 2000;
const COLUMNS: [&str; 6] = [
    "copy",
    "selection",
    "insertion",
    "index",
    "unstable",
    "unstbl idx",
];

struct Measured {
    size: usize,
    sel: Counts,
    ins: Counts,
    idx: Counts,
    ok: bool,
    t: [f64; 6], // copy, selection, insertion, index, sort_unstable_by, unstable idx
}

// measure runs everything for one record size.
fn measure<const P: usize>(keys: &[usize]) -> Measured {
    let mut pristine = vec![
        Rec::<P> {
            payload: [0; P],
            key: 0
        };
        N
    ];
    for (r, &k) in pristine.iter_mut().zip(keys) {
        r.key = k as i64;
    }
    let mut work = pristine.clone();
    let mut idx = vec![0usize; N];
    let by_key = |a: &Rec<P>, b: &Rec<P>| a.key.cmp(&b.key);

    // The keys are a permutation of 0..n-1, so "sorted" means work[i].key == i
    // for every i -- a lost or duplicated record would show up here.
    let sorted = |w: &[Rec<P>]| w.iter().enumerate().all(|(i, r)| r.key == i as i64);
    work.copy_from_slice(&pristine);
    let sel = selection_sort(&mut work);
    let mut ok = sorted(&work);
    work.copy_from_slice(&pristine);
    let ins = insertion_sort(&mut work);
    ok &= sorted(&work);
    work.copy_from_slice(&pristine);
    let ix = index_sort(&mut work, &mut idx);
    ok &= sorted(&work);
    work.copy_from_slice(&pristine);
    work.sort_unstable_by(by_key);
    ok &= sorted(&work);
    work.copy_from_slice(&pristine);
    std_index_sort(&mut work, &mut idx);
    ok &= sorted(&work);

    // Timings: a sort MUTATES its input, so every timed iteration first copies
    // the pristine input back; the "copy" column times that copy alone.
    let t = [
        ns_per_op(|| work.copy_from_slice(black_box(&pristine))),
        ns_per_op(|| {
            work.copy_from_slice(&pristine);
            black_box(selection_sort(&mut work));
        }),
        ns_per_op(|| {
            work.copy_from_slice(&pristine);
            black_box(insertion_sort(&mut work));
        }),
        ns_per_op(|| {
            work.copy_from_slice(&pristine);
            black_box(index_sort(&mut work, &mut idx));
        }),
        ns_per_op(|| {
            work.copy_from_slice(&pristine);
            work.sort_unstable_by(by_key);
            black_box(&work);
        }),
        ns_per_op(|| {
            work.copy_from_slice(&pristine);
            std_index_sort(&mut work, &mut idx);
            black_box(&work);
        }),
    ];
    Measured {
        size: size_of::<Rec<P>>(),
        sel,
        ins,
        idx: ix,
        ok,
        t,
    }
}

fn main() {
    // keys: a random permutation of 0..n-1, Fisher-Yates on a fixed PCG seed
    let mut rng = Pcg::new(15, 16);
    let mut keys: Vec<usize> = (0..N).collect();
    for i in (1..N).rev() {
        let j = rng.int_n(i as u64 + 1) as usize;
        keys.swap(i, j);
    }
    let inv: usize = (0..N)
        .map(|i| keys[i + 1..].iter().filter(|&&k| keys[i] > k).count())
        .sum();
    println!(
        "n = {N} records, keys = a PCG permutation with I = {inv} inversions (n(n-1)/4 = {})",
        N * (N - 1) / 4
    );

    let rs = [
        measure::<0>(&keys),
        measure::<56>(&keys),
        measure::<248>(&keys),
        measure::<1016>(&keys),
    ];
    let same = rs
        .iter()
        .all(|r| r.ok && r.sel == rs[0].sel && r.ins == rs[0].ins && r.idx == rs[0].idx);
    println!(
        "record sizes {}, {}, {}, {} bytes: every sort correct, counts identical at every size: {same}",
        rs[0].size, rs[1].size, rs[2].size, rs[3].size
    );

    println!();
    println!(
        "operation counts (deterministic; a write = one slot assignment, see the top comment)"
    );
    println!();
    println!(
        "  {:<16} {:>11} {:>10} {:>10} {:>9} {:>9} {:>9} {:>9}",
        "",
        "comparisons",
        "rec writes",
        "idx writes",
        "MB @ 8B",
        "MB @ 64B",
        "MB @ 256B",
        "MB @ 1KiB"
    );
    let row = |name: &str, c: Counts| {
        print!(
            "  {name:<16} {:>11} {:>10} {:>10}",
            c.comps, c.rec_writes, c.idx_writes
        );
        for r in &rs {
            let bytes = c.rec_writes * r.size + c.idx_writes * 8;
            print!(" {:>9.3}", bytes as f64 / 1e6);
        }
        println!();
    };
    let (s, ins, ix) = (rs[0].sel, rs[0].ins, rs[0].idx);
    row("selection", s);
    row("insertion", ins);
    row("index + permute", ix);
    println!();
    println!(
        "selection: {} comparisons = n(n-1)/2: {}; {} swaps (at most n-1 = {})",
        s.comps,
        s.comps == N * (N - 1) / 2,
        s.rec_writes / 2,
        N - 1
    );
    println!(
        "insertion: {} record writes = {} shifts + {} drops; shifts == I: {}",
        ins.rec_writes,
        ins.shifts,
        ins.rec_writes - ins.shifts,
        ins.shifts == inv
    );
    println!(
        "insertion vs selection: {:.2}x the comparisons, {:.0}x the record writes",
        ins.comps as f64 / s.comps as f64,
        ins.rec_writes as f64 / s.rec_writes as f64
    );
    println!(
        "index sort: the same comparisons as insertion: {}; its {} shifts are of 8-byte",
        ix.comps == ins.comps,
        ix.shifts
    );
    println!(
        "  indices, and the permutation pass makes {} record writes (n = {N})",
        ix.rec_writes
    );

    println!();
    println!("time per sort, microseconds (best of 3 rounds; every iteration re-copies the");
    println!("pristine input; 'copy' is that copy alone and is included in every other column)");
    println!("(unstable = sort_unstable_by on the records, unstbl idx = the same on an index Vec)");
    println!();
    print!("  {:<7}", "record");
    for name in COLUMNS {
        print!(" {name:>10}");
    }
    println!("  fastest");
    for r in &rs {
        print!("  {:<7}", format!("{}B", r.size));
        let mut best = 1;
        for (i, &t) in r.t.iter().enumerate() {
            print!(" {:>10.1}", t / 1e3);
            if i > 0 && t < r.t[best] {
                best = i;
            }
        }
        println!("  {}", COLUMNS[best]);
    }

    println!();
    match rs.iter().find(|r| r.t[1] < r.t[2]) {
        None => println!("insertion beat selection at every record size in this run"),
        Some(r) => println!(
            "selection first beat insertion at {}-byte records in this run",
            r.size
        ),
    }
    for r in &rs {
        println!(
            "  {:>4}B: insertion/selection {:>5.2}x   insertion/index {:>5.2}x   unstable rec/idx {:>5.2}x",
            r.size,
            r.t[2] / r.t[1],
            r.t[2] / r.t[3],
            r.t[4] / r.t[5]
        );
    }
    println!();
    println!("the counts say why: the comparisons do not depend on the record size, the bytes");
    println!("moved do. Sorting indices keeps an algorithm's comparisons and replaces its record");
    println!("moves with 8-byte moves plus at most n record writes -- for the Theta(n^2) sort");
    println!("and for sort_unstable_by alike (whose comparator gets references, not copies).");
}
```

**Rust sample output:**

```
n = 2000 records, keys = a PCG permutation with I = 986319 inversions (n(n-1)/4 = 999500)
record sizes 8, 64, 256, 1024 bytes: every sort correct, counts identical at every size: true

operation counts (deterministic; a write = one slot assignment, see the top comment)

                   comparisons rec writes idx writes   MB @ 8B  MB @ 64B MB @ 256B MB @ 1KiB
  selection            1999000       3990          0     0.032     0.255     1.021     4.086
  insertion             988308     988314          0     7.907    63.252   253.008  1012.034
  index + permute       988308       2000     992314     7.955     8.067     8.451     9.987

selection: 1999000 comparisons = n(n-1)/2: true; 1995 swaps (at most n-1 = 1999)
insertion: 988314 record writes = 986319 shifts + 1995 drops; shifts == I: true
insertion vs selection: 0.49x the comparisons, 248x the record writes
index sort: the same comparisons as insertion: true; its 986319 shifts are of 8-byte
  indices, and the permutation pass makes 2000 record writes (n = 2000)

time per sort, microseconds (best of 3 rounds; every iteration re-copies the
pristine input; 'copy' is that copy alone and is included in every other column)
(unstable = sort_unstable_by on the records, unstbl idx = the same on an index Vec)

  record        copy  selection  insertion      index   unstable unstbl idx  fastest
  8B             0.2     1027.6      271.5      392.9        9.7       18.0  unstable
  64B            1.7     1026.9      583.0      398.0       36.3       21.1  unstbl idx
  256B           6.9     1186.8     3278.8      560.9       53.3       34.0  unstbl idx
  1024B         30.4     1341.7    14929.1      694.8      270.8       98.6  unstbl idx

selection first beat insertion at 256-byte records in this run
     8B: insertion/selection  0.26x   insertion/index  0.69x   unstable rec/idx  0.54x
    64B: insertion/selection  0.57x   insertion/index  1.47x   unstable rec/idx  1.72x
   256B: insertion/selection  2.76x   insertion/index  5.85x   unstable rec/idx  1.57x
  1024B: insertion/selection 11.13x   insertion/index 21.49x   unstable rec/idx  2.75x

the counts say why: the comparisons do not depend on the record size, the bytes
moved do. Sorting indices keeps an algorithm's comparisons and replaces its record
moves with 8-byte moves plus at most n record writes -- for the Theta(n^2) sort
and for sort_unstable_by alike (whose comparator gets references, not copies).
```

> **Go vs Rust:** The counts match Go's line for line, because the PCG keys and the algorithms are the same. In Rust too, selection overtook insertion at 256 B in every run (roughly 3× there, ≈ 10–12× at 1 KiB). The stdlib column differs. On 1 KiB records `sort_unstable_by` took ≈ 0.3 ms against ≈ 1.9 ms for Go's `SortFunc`, and it was only about 3× slower than the same sort on an index `Vec` (Go: ≈ 13–14×). A plausible reason, not isolated here: a Rust comparator gets `&T`, while Go's `func(a, b E) int` gets copies of both records.
> One change was forced by Rust's code generation: `if a[j].key < a[m].key { m = j }` compiled to a `csel` whose result feeds the next load, and that loop ran ≈ 3.4× slower (3.4 ms against 1.0 ms at 8 B). So both versions keep the minimum key in a local. Go's compiler emits a branch for the same line (checked with `go build -gcflags=-S`).

**Complexity:** selection Θ(n²) comparisons and Θ(n) moves (1,999,000 and 3,990 writes here) · insertion Θ(n²) of both (988,308 and 988,314; shifts = I exactly) · index sort: insertion's comparisons, Θ(n²) moves of 8-byte indices, then ≤ n record writes · bytes moved, not comparisons, decide large-record sorts: at 1 KiB insertion moves 1,012 MB against selection's 4 MB. In every run selection overtook insertion at 256-byte records (≈ 2.6× faster there, ≈ 10–11× at 1 KiB). `slices.SortFunc` on an index slice was the fastest sort from 64 B up, ≈ 13–14× faster than `SortFunc` on the 1 KiB records themselves.

---

## 16. Capstone: `InsertionSortFunc`, built, proved, measured

`🔴 hard` · *All three passes*

This example packages the lesson's sort the way the standard library would. It builds a generic, stable,
shift-based `InsertionSortFunc` and a counted variant beside it. It then proves both against
`slices.SortStableFunc` on 100,000 PCG cases with a narrow key range, so ties are everywhere and stability is
actually tested. The inversion identities are proved as properties (shifts = I, I ≤ comparisons ≤ I + n − 1), the
loop invariant is checked after every outer step, and a planted `<=` mutant shows that the harness can fail.
Finally it measures the sort against both stdlib sorts at small n and on a nearly sorted n = 100,000. The
measurement surfaced something that was not in the plan. Called directly, the small generic function is inlined
and Go then calls `cmp.Compare` directly. So a `//go:noinline` column is needed to compare the algorithms rather
than the inlining. The sort is Θ(n + I) time and Θ(1) space, and the program shows it makes no allocations.

**Steps:**

1. Write `InsertionSortFunc` with a strict `< 0`, and `InsertionSortCounted` with the same comparisons in the same
   order. Give the counted version an internal per-step hook.
2. Generate records `(key in [0,4), tag = index)` and compare whole records with the stable oracle. Check
   shifts and comparisons against brute-force I, then run the `<=` mutant through the same harness.
3. Time own, own behind `//go:noinline`, `slices.SortFunc` and `slices.SortStableFunc`. Read the ratios where
   the stdlib runs the same algorithm (n ≤ 12), at n = 64, and on the 4-sorted input.

```go
package main

import (
	"cmp"
	"fmt"
	"math"
	"math/rand/v2"
	"slices"
	"testing"
	"time"
)

// ---------------------------------------------------------------- BUILT --

// InsertionSortFunc sorts s in place in ascending order as determined by cmp,
// which must be a strict weak ordering (as for slices.SortFunc). It is stable:
// an element moves left only past elements that compare strictly greater.
// Theta(n + I) time for I inversions, Theta(1) extra space, no allocations.
func InsertionSortFunc[E any](s []E, cmp func(a, b E) int) {
	for i := 1; i < len(s); i++ {
		x := s[i] // hold it; s[i] is now a hole
		j := i
		for j > 0 && cmp(x, s[j-1]) < 0 { // strict < 0: equal keys never pass each other
			s[j] = s[j-1] // shift right; the hole moves left
			j--
		}
		s[j] = x
	}
}

type Counts struct{ Comparisons, Shifts int }

// InsertionSortCounted is InsertionSortFunc with counters: the same comparisons
// in the same order, so its counts describe the uncounted version too.
func InsertionSortCounted[E any](s []E, cmp func(a, b E) int) Counts {
	return insertionSortCounted(s, cmp, nil)
}

// insertionSortCounted calls after(s, i) once s[i] has been inserted, i.e. when
// s[:i+1] should be sorted -- the hook the invariant check uses.
func insertionSortCounted[E any](s []E, cmp func(a, b E) int, after func(s []E, i int)) Counts {
	var c Counts
	for i := 1; i < len(s); i++ {
		x := s[i]
		j := i
		for j > 0 {
			c.Comparisons++
			if cmp(x, s[j-1]) >= 0 {
				break
			}
			s[j] = s[j-1]
			c.Shifts++
			j--
		}
		s[j] = x
		if after != nil {
			after(s, i)
		}
	}
	return c
}

// mutantSort is the one-character bug of this lesson: <= 0 instead of < 0.
func mutantSort[E any](s []E, cmp func(a, b E) int) {
	for i := 1; i < len(s); i++ {
		x := s[i]
		j := i
		for j > 0 && cmp(x, s[j-1]) <= 0 { // BUG: walks past EQUAL keys too
			s[j] = s[j-1]
			j--
		}
		s[j] = x
	}
}

// --------------------------------------------------------------- PROVED --

// Rec: a key from a NARROW range (so ties are everywhere) and a tag = its input
// index (so we can see whether equal keys kept their order).
type Rec struct{ Key, Tag int }

func byKey(a, b Rec) int { return cmp.Compare(a.Key, b.Key) }

func prove() {
	const cases = 100_000
	rng := rand.New(rand.NewPCG(16, 17))
	var badA, badA2, badB, badC, withTies, totC, totS int
	steps, badD := 0, 0
	mutSorted, mutEqual := 0, 0
	var mutSmallest []Rec

	for t := 0; t < cases; t++ {
		n := rng.IntN(41)
		orig := make([]Rec, n)
		for i := range orig {
			orig[i] = Rec{Key: rng.IntN(4), Tag: i}
		}
		want := slices.Clone(orig) // the oracle: the stdlib's stable sort
		slices.SortStableFunc(want, byKey)
		inv := 0 // inversions by brute force: pairs i < j with key[i] > key[j]
		for i := range orig {
			for j := i + 1; j < n; j++ {
				if orig[i].Key > orig[j].Key {
					inv++
				}
			}
		}
		for i := 1; i < n; i++ { // want is sorted, so equal keys are adjacent
			if want[i].Key == want[i-1].Key {
				withTies++
				break
			}
		}

		got := slices.Clone(orig)
		InsertionSortFunc(got, byKey)
		if !slices.Equal(got, want) { // == on the WHOLE record: Key and Tag
			badA++
		}

		got = slices.Clone(orig)
		var c Counts
		if t < 1000 { // (d) the invariant after every outer step
			c = insertionSortCounted(got, byKey, func(s []Rec, i int) {
				steps++
				prefix := slices.Clone(orig[:i+1])
				slices.SortStableFunc(prefix, byKey)
				if !slices.Equal(s[:i+1], prefix) || !slices.Equal(s[i+1:], orig[i+1:]) {
					badD++
				}
			})
		} else {
			c = InsertionSortCounted(got, byKey)
		}
		if !slices.Equal(got, want) {
			badA2++
		}
		if c.Shifts != inv {
			badB++
		}
		if c.Comparisons < inv || c.Comparisons > inv+max(n-1, 0) {
			badC++
		}
		totC += c.Comparisons
		totS += c.Shifts

		mut := slices.Clone(orig)
		mutantSort(mut, byKey)
		if slices.IsSortedFunc(mut, byKey) {
			mutSorted++
		}
		if slices.Equal(mut, want) {
			mutEqual++
		} else if mutSmallest == nil || n < len(mutSmallest) {
			mutSmallest = orig
		}
	}

	fmt.Printf("PROVED  %d PCG cases, n in [0, 40], keys in [0, 4), tag = input index\n", cases)
	fmt.Printf("  (a) InsertionSortFunc    == slices.SortStableFunc, whole records: %d mismatches\n", badA)
	fmt.Printf("      InsertionSortCounted == slices.SortStableFunc, whole records: %d mismatches\n", badA2)
	fmt.Printf("  (b) Shifts == brute-force inversions I:                         %d mismatches\n", badB)
	fmt.Printf("  (c) I <= Comparisons <= I + n - 1:                              %d violations\n", badC)
	fmt.Printf("  (d) invariant after each of %d outer steps (first 1000 cases):  %d violations\n", steps, badD)
	fmt.Printf("      (prefix == stable sort of the original prefix, suffix untouched)\n")
	fmt.Printf("  cases containing equal keys: %d of %d, so stability was exercised, not assumed\n", withTies, cases)
	fmt.Printf("  totals: %d comparisons, %d shifts (= the total of I)\n", totC, totS)
	fmt.Println()
	fmt.Println("  does the harness have teeth? the mutant with <= 0 instead of < 0:")
	fmt.Printf("    sorted by key:          %d of %d   <- a sortedness test passes it\n", mutSorted, cases)
	fmt.Printf("    equal to the oracle:    %d of %d   <- the stability check fails it\n", mutEqual, cases)
	m := slices.Clone(mutSmallest)
	mutantSort(m, byKey)
	fmt.Printf("    smallest failing input (key, tag): %v -> %v\n", mutSmallest, m)
}

func edgeCases() {
	recs := func(keys ...int) []Rec {
		rs := make([]Rec, len(keys))
		for i, k := range keys {
			rs[i] = Rec{k, i}
		}
		return rs
	}
	cases := []struct {
		name       string
		in         []Rec
		comps, shf int
	}{
		{"empty", recs(), 0, 0},
		{"one element", recs(7), 0, 0},
		{"all equal, n=10", recs(5, 5, 5, 5, 5, 5, 5, 5, 5, 5), 9, 0},
		{"sorted, n=10", recs(0, 1, 2, 3, 4, 5, 6, 7, 8, 9), 9, 0},
		{"reversed, n=10", recs(9, 8, 7, 6, 5, 4, 3, 2, 1, 0), 45, 45},
		{"1, 0, 1 (a tie)", recs(1, 0, 1), 2, 1},
	}
	fmt.Println()
	fmt.Println("edge cases, exact expected counts (want = derived by hand, not by running it):")
	fmt.Printf("  %-20s %12s %12s  %s\n", "input", "want c/s", "got c/s", "ok")
	for _, tc := range cases {
		got := slices.Clone(tc.in)
		c := InsertionSortCounted(got, byKey)
		want := slices.Clone(tc.in)
		slices.SortStableFunc(want, byKey)
		ok := c.Comparisons == tc.comps && c.Shifts == tc.shf && slices.Equal(got, want)
		fmt.Printf("  %-20s %12s %12s  %v\n", tc.name, fmt.Sprintf("%d/%d", tc.comps, tc.shf),
			fmt.Sprintf("%d/%d", c.Comparisons, c.Shifts), ok)
	}
}

// ------------------------------------------------------------- MEASURED --

// nsPerOp times each f in fs and returns ns per call of each: best of 12
// rounds, where a round gives every f in turn a batch that doubles until it
// runs >= 5 ms. Interleaving the rounds spreads a burst of machine noise over
// all columns instead of one. f(k) does iteration k and restores its own input.
func nsPerOp(fs ...func(k int)) []float64 {
	best := make([]float64, len(fs))
	for i := range best {
		best[i] = math.MaxFloat64
	}
	for round := 0; round < 12; round++ {
		for i, f := range fs {
			for batch := 1; ; batch *= 2 {
				start := time.Now()
				for k := 0; k < batch; k++ {
					f(k)
				}
				if el := time.Since(start); el >= 5*time.Millisecond {
					best[i] = min(best[i], float64(el.Nanoseconds())/float64(batch))
					break
				}
			}
		}
	}
	return best
}

// ownOpaque hides InsertionSortFunc from the inliner. Called directly it is small
// enough to be INLINED, and then Go calls (and inlines) cmp.Compare directly --
// checked with go build -gcflags=-m. slices.SortFunc is too big for that and calls
// cmp through the func value, and so does ownOpaque: algorithms, not inlining.
//
//go:noinline
func ownOpaque(s []int, cmp func(a, b int) int) { InsertionSortFunc(s, cmp) }

func measure() {
	rng := rand.New(rand.NewPCG(16, 18))
	icmp := cmp.Compare[int]
	type sorter struct {
		name string
		sort func([]int)
	}
	sorters := []sorter{
		{"own", func(s []int) { InsertionSortFunc(s, icmp) }},
		{"own noinl", func(s []int) { ownOpaque(s, icmp) }},
		{"SortFunc", func(s []int) { slices.SortFunc(s, icmp) }},
		{"SortStable", func(s []int) { slices.SortStableFunc(s, icmp) }},
	}

	fmt.Println()
	fmt.Println("MEASURED  ns per sort of []int with cmp.Compare[int] (best of 12 rounds; every")
	fmt.Println("iteration copies a pristine input first, and 'copy' times that copy alone)")
	fmt.Println()
	fmt.Printf("  %-18s %7s %9s %10s %10s %10s  %s\n", "input", "copy", "own", "own noinl",
		"SortFunc", "SortStable", "fastest")
	type timing struct {
		n int
		t []float64 // copy, then one per sorter
	}
	var rows []timing
	row := func(label string, pool [][]int) {
		n := len(pool[0])
		work := make([]int, n)
		fs := []func(int){func(k int) { copy(work, pool[k%len(pool)]) }}
		for _, s := range sorters {
			fs = append(fs, func(k int) { copy(work, pool[k%len(pool)]); s.sort(work) })
		}
		t := nsPerOp(fs...)
		best := 1
		for i := 2; i < len(t); i++ {
			if t[i] < t[best] {
				best = i
			}
		}
		fmt.Printf("  %-18s %7.0f %9.0f %10.0f %10.0f %10.0f  %s\n", label, t[0], t[1], t[2], t[3], t[4],
			sorters[best-1].name)
		rows = append(rows, timing{n, t})
	}
	// small n: a pool of 1000 different permutations, so the branch predictor
	// cannot memorise one input
	for _, n := range []int{4, 8, 12, 16, 20, 32, 64} {
		pool := make([][]int, 1000)
		for i := range pool { // Fisher-Yates with IntN, so the Rust twin can replay it
			pool[i] = make([]int, n)
			for v := range pool[i] {
				pool[i][v] = v
			}
			for v := n - 1; v > 0; v-- {
				j := rng.IntN(v + 1)
				pool[i][v], pool[i][j] = pool[i][j], pool[i][v]
			}
		}
		row(fmt.Sprintf("random, n=%d", n), pool)
	}

	// large n, nearly sorted: shuffle each block of k+1 = 5 consecutive values,
	// so every element is at most k = 4 places from home
	const big, k = 100_000, 4
	ks := make([]int, big)
	for i := range ks {
		ks[i] = i
	}
	for b := 0; b < big; b += k + 1 {
		blk := ks[b : b+k+1]
		for i := k; i > 0; i-- {
			j := rng.IntN(i + 1)
			blk[i], blk[j] = blk[j], blk[i]
		}
	}
	row("4-sorted, n=100000", [][]int{ks})

	// read the table: computed from this run's numbers, not asserted. Up to
	// n = 12 SortFunc runs the same algorithm as own noinl (swap version, same
	// indirect calls), so there their ratio isolates shift vs swap.
	fmt.Println()
	lo, hi, logSum := math.MaxFloat64, 0.0, 0.0
	fmt.Print("  SortFunc / own noinl:")
	for _, r := range rows {
		lo, hi = min(lo, r.t[2]/r.t[1]), max(hi, r.t[2]/r.t[1])
		if r.n <= 64 { // the random rows
			fmt.Printf(" n=%d %.2f", r.n, r.t[3]/r.t[2])
		}
		if r.n <= 12 {
			logSum += math.Log(r.t[3] / r.t[2])
		}
	}
	fmt.Printf("\n  geometric mean of that ratio over n = 4, 8, 12 (same algorithm in both): %.2f\n",
		math.Exp(logSum/3))
	fmt.Printf("  inlining alone (own noinl / own) was worth %.1fx to %.1fx across the rows\n", lo, hi)
	last := rows[len(rows)-1].t
	fmt.Printf("  4-sorted n=100000: SortFunc took %.1fx and SortStableFunc %.1fx the time of own noinl\n",
		last[3]/last[2], last[4]/last[2])

	// what each sort pays on that input, in comparator calls
	calls := 0
	counting := func(a, b int) int { calls++; return cmp.Compare(a, b) }
	work := slices.Clone(ks)
	c := InsertionSortCounted(work, icmp)
	copy(work, ks)
	slices.SortFunc(work, counting)
	fmt.Printf("  on the 4-sorted input own made %d shifts (= I) and %d comparisons;\n", c.Shifts, c.Comparisons)
	fmt.Printf("  comparator calls there: SortFunc %d, ", calls)
	calls = 0
	copy(work, ks)
	slices.SortStableFunc(work, counting)
	fmt.Printf("SortStableFunc %d\n", calls)

	// allocations: the claim "Theta(1) space, no allocations", checked
	fmt.Print("  allocations per sort of it (testing.AllocsPerRun):")
	for _, s := range sorters {
		a := testing.AllocsPerRun(20, func() { copy(work, ks); s.sort(work) })
		fmt.Printf(" %s %.0f", s.name, a)
	}
	fmt.Println()
}

func main() {
	prove()
	edgeCases()
	measure()
	fmt.Println()
	fmt.Println("from the source: at n <= 12 slices.SortFunc runs the SWAP version of this same")
	fmt.Println("algorithm, and at n <= 20 slices.SortStableFunc runs insertion sort on one block,")
	fmt.Println("so up to there own noinl vs the stdlib compares shift vs swap, not algorithms.")
	fmt.Println("the own sort is Theta(n + I): it wins where I is small, tiny or nearly sorted input.")
	fmt.Println("On large random input I ~ n^2/4, and lesson 01 example 11 timed where it loses.")
}
```

**Sample output:**

```
PROVED  100000 PCG cases, n in [0, 40], keys in [0, 4), tag = input index
  (a) InsertionSortFunc    == slices.SortStableFunc, whole records: 0 mismatches
      InsertionSortCounted == slices.SortStableFunc, whole records: 0 mismatches
  (b) Shifts == brute-force inversions I:                         0 mismatches
  (c) I <= Comparisons <= I + n - 1:                              0 violations
  (d) invariant after each of 18591 outer steps (first 1000 cases):  0 violations
      (prefix == stable sort of the original prefix, suffix untouched)
  cases containing equal keys: 92046 of 100000, so stability was exercised, not assumed
  totals: 11525644 comparisons, 9724266 shifts (= the total of I)

  does the harness have teeth? the mutant with <= 0 instead of < 0:
    sorted by key:          100000 of 100000   <- a sortedness test passes it
    equal to the oracle:    7954 of 100000   <- the stability check fails it
    smallest failing input (key, tag): [{3 0} {3 1}] -> [{3 1} {3 0}]

edge cases, exact expected counts (want = derived by hand, not by running it):
  input                    want c/s      got c/s  ok
  empty                         0/0          0/0  true
  one element                   0/0          0/0  true
  all equal, n=10               9/0          9/0  true
  sorted, n=10                  9/0          9/0  true
  reversed, n=10              45/45        45/45  true
  1, 0, 1 (a tie)               2/1          2/1  true

MEASURED  ns per sort of []int with cmp.Compare[int] (best of 12 rounds; every
iteration copies a pristine input first, and 'copy' times that copy alone)

  input                 copy       own  own noinl   SortFunc SortStable  fastest
  random, n=4              2         7         16         15         15  own
  random, n=8              2        18         47         46         46  own
  random, n=12             3        30         99         98         95  own
  random, n=16             3        44        164        156        167  own
  random, n=20             4        65        237        229        256  own
  random, n=32             5       190        512        518        569  own
  random, n=64            10       692       1685       1437       1697  own
  4-sorted, n=100000   10016    406607     673953    2720312     748646  own

  SortFunc / own noinl: n=4 0.97 n=8 0.98 n=12 0.99 n=16 0.95 n=20 0.97 n=32 1.01 n=64 0.85
  geometric mean of that ratio over n = 4, 8, 12 (same algorithm in both): 0.98
  inlining alone (own noinl / own) was worth 1.7x to 3.7x across the rows
  4-sorted n=100000: SortFunc took 4.0x and SortStableFunc 1.1x the time of own noinl
  on the 4-sorted input own made 100419 shifts (= I) and 200417 comparisons;
  comparator calls there: SortFunc 1571768, SortStableFunc 214209
  allocations per sort of it (testing.AllocsPerRun): own 0 own noinl 0 SortFunc 0 SortStable 0

from the source: at n <= 12 slices.SortFunc runs the SWAP version of this same
algorithm, and at n <= 20 slices.SortStableFunc runs insertion sort on one block,
so up to there own noinl vs the stdlib compares shift vs swap, not algorithms.
the own sort is Theta(n + I): it wins where I is small, tiny or nearly sorted input.
On large random input I ~ n^2/4, and lesson 01 example 11 timed where it loses.
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::cmp::Ordering;
use std::fmt::Write as _;
use std::hint::black_box;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};
use std::time::{Duration, Instant};

// ---------------------------------------------------------------- BUILT --

/// Sorts `v` in place, ascending as determined by `f`, which must be a total
/// order (as for `sort_by`). Stable: an element moves left only past elements
/// that compare strictly Greater. Theta(n + I) time for I inversions,
/// Theta(1) extra space, no allocations. Works for any T, Copy or not.
///
/// Go holds x = s[i] in a variable and shifts. Safe Rust cannot move v[i] out
/// of the slice and leave a hole (std's insert_tail does exactly that with
/// `unsafe` ptr::read + ptr::copy_nonoverlapping). So: first find the slot j by
/// comparing v[i] against v[i-1],
/// v[i-2], ... -- nothing has moved yet, so these are exactly the comparisons
/// Go makes, in the same order -- then `rotate_right(1)` the run v[j..=i]: v[i]
/// lands at j and the run shifts right by one, a memmove.
fn insertion_sort_by<T, F: FnMut(&T, &T) -> Ordering>(v: &mut [T], mut f: F) {
    for i in 1..v.len() {
        let mut j = i;
        while j > 0 && f(&v[i], &v[j - 1]) == Ordering::Less {
            // strict Less: equal keys never pass each other
            j -= 1;
        }
        v[j..=i].rotate_right(1);
    }
}

/// The other safe spelling: walk v[i] left by adjacent swaps (Go's stdlib
/// insertion sort is written this way). f(&v[j], &v[j-1]) compares the moving
/// element with the same neighbours in the same order, so the comparisons are
/// again identical -- only the writes differ (a swap is 3 moves, not 1).
fn insertion_sort_by_swaps<T, F: FnMut(&T, &T) -> Ordering>(v: &mut [T], mut f: F) {
    for i in 1..v.len() {
        let mut j = i;
        while j > 0 && f(&v[j], &v[j - 1]) == Ordering::Less {
            v.swap(j, j - 1);
            j -= 1;
        }
    }
}

#[derive(Clone, Copy, Default)]
struct Counts {
    comparisons: usize,
    shifts: usize,
}

/// insertion_sort_by with counters: the same comparisons in the same order, so
/// its counts describe the uncounted version too.
fn insertion_sort_counted<T, F: FnMut(&T, &T) -> Ordering>(v: &mut [T], f: F) -> Counts {
    insertion_sort_counted_with(v, f, |_, _| {})
}

/// calls after(v, i) once v[i] has been inserted, i.e. when v[..=i] should be
/// sorted -- the hook the invariant check uses. It gets a shared borrow of the
/// slice we hold mutably: fine, because the borrow ends before we write again.
fn insertion_sort_counted_with<T, F, H>(v: &mut [T], mut f: F, mut after: H) -> Counts
where
    F: FnMut(&T, &T) -> Ordering,
    H: FnMut(&[T], usize),
{
    let mut c = Counts::default();
    for i in 1..v.len() {
        let mut j = i;
        while j > 0 {
            c.comparisons += 1;
            if f(&v[i], &v[j - 1]) != Ordering::Less {
                break;
            }
            j -= 1;
        }
        c.shifts += i - j;
        v[j..=i].rotate_right(1);
        after(v, i);
    }
    c
}

/// The one-character bug of this lesson: `!= Greater` (<=) instead of `== Less`.
fn mutant_sort<T, F: FnMut(&T, &T) -> Ordering>(v: &mut [T], mut f: F) {
    for i in 1..v.len() {
        let mut j = i;
        while j > 0 && f(&v[i], &v[j - 1]) != Ordering::Greater {
            // BUG: walks past EQUAL keys too
            j -= 1;
        }
        v[j..=i].rotate_right(1);
    }
}

// --------------------------------------------------------------- PROVED --

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM) and its IntN, so both
// programs test exactly the same 100,000 inputs.
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

// Rec: a key from a NARROW range (so ties are everywhere) and a tag = its input
// index (so we can see whether equal keys kept their order). Deliberately NOT
// Copy: the sort must work by moving, and the oracle by cloning.
#[derive(Clone, Debug, PartialEq)]
struct Rec {
    key: u8,
    tag: usize,
}

fn by_key(a: &Rec, b: &Rec) -> Ordering {
    a.key.cmp(&b.key)
}

/// Go's %v of a []Rec: [{3 0} {3 1}]
fn go_recs(rs: &[Rec]) -> String {
    let parts: Vec<String> = rs
        .iter()
        .map(|r| format!("{{{} {}}}", r.key, r.tag))
        .collect();
    format!("[{}]", parts.join(" "))
}

fn prove() {
    const CASES: usize = 100_000;
    let mut rng = Pcg::new(16, 17);
    let (mut bad_a, mut bad_a2, mut bad_b, mut bad_c, mut with_ties) = (0, 0, 0, 0, 0);
    let (mut tot_c, mut tot_s, mut steps, mut bad_d, mut bad_a3) = (0, 0, 0, 0, 0);
    let (mut mut_sorted, mut mut_equal) = (0, 0);
    let mut mut_smallest: Option<Vec<Rec>> = None;

    for t in 0..CASES {
        let n = rng.int_n(41);
        let orig: Vec<Rec> = (0..n)
            .map(|tag| Rec {
                key: rng.int_n(4) as u8,
                tag,
            })
            .collect();
        let mut want = orig.clone(); // the oracle: std's stable sort
        want.sort_by(by_key);
        // inversions by brute force: pairs i < j with key[i] > key[j]
        let inv: usize = (0..n)
            .map(|i| orig[i + 1..].iter().filter(|r| orig[i].key > r.key).count())
            .sum();
        if want.windows(2).any(|w| w[0].key == w[1].key) {
            with_ties += 1; // want is sorted, so equal keys are adjacent
        }

        let mut got = orig.clone();
        insertion_sort_by(&mut got, by_key);
        if got != want {
            // == on the WHOLE record: key and tag
            bad_a += 1;
        }

        let mut got = orig.clone();
        insertion_sort_by_swaps(&mut got, by_key);
        bad_a3 += usize::from(got != want);

        let mut got = orig.clone();
        let c = if t < 1000 {
            // (d) the invariant after every outer step
            insertion_sort_counted_with(&mut got, by_key, |s, i| {
                steps += 1;
                let mut prefix = orig[..=i].to_vec();
                prefix.sort_by(by_key);
                if s[..=i] != prefix[..] || s[i + 1..] != orig[i + 1..] {
                    bad_d += 1;
                }
            })
        } else {
            insertion_sort_counted(&mut got, by_key)
        };
        bad_a2 += usize::from(got != want);
        bad_b += usize::from(c.shifts != inv);
        bad_c += usize::from(c.comparisons < inv || c.comparisons > inv + n.saturating_sub(1));
        tot_c += c.comparisons;
        tot_s += c.shifts;

        let mut m = orig.clone();
        mutant_sort(&mut m, by_key);
        mut_sorted += usize::from(m.is_sorted_by(|a, b| a.key <= b.key));
        if m == want {
            mut_equal += 1;
        } else if mut_smallest.as_ref().is_none_or(|s| n < s.len()) {
            mut_smallest = Some(orig);
        }
    }

    println!("PROVED  {CASES} PCG cases, n in [0, 40], keys in [0, 4), tag = input index");
    println!(
        "  (a) insertion_sort_by      == sort_by (stable), whole records:  {bad_a} mismatches"
    );
    println!(
        "      insertion_sort_counted == sort_by (stable), whole records:  {bad_a2} mismatches"
    );
    println!(
        "      insertion_sort_by_swaps == sort_by (stable), whole records: {bad_a3} mismatches"
    );
    println!(
        "  (b) shifts == brute-force inversions I:                         {bad_b} mismatches"
    );
    println!(
        "  (c) I <= comparisons <= I + n - 1:                              {bad_c} violations"
    );
    println!(
        "  (d) invariant after each of {steps} outer steps (first 1000 cases):  {bad_d} violations"
    );
    println!("      (prefix == stable sort of the original prefix, suffix untouched)");
    println!(
        "  cases containing equal keys: {with_ties} of {CASES}, so stability was exercised, not assumed"
    );
    println!("  totals: {tot_c} comparisons, {tot_s} shifts (= the total of I)");
    println!();
    println!("  does the harness have teeth? the mutant with != Greater instead of == Less:");
    println!(
        "    sorted by key:          {mut_sorted} of {CASES}   <- a sortedness test passes it"
    );
    println!(
        "    equal to the oracle:    {mut_equal} of {CASES}   <- the stability check fails it"
    );
    let small = mut_smallest.expect("the mutant failed at least once");
    let mut m = small.clone();
    mutant_sort(&mut m, by_key);
    println!(
        "    smallest failing input (key, tag): {} -> {}",
        go_recs(&small),
        go_recs(&m)
    );
}

fn edge_cases() {
    let recs = |keys: &[u8]| -> Vec<Rec> {
        keys.iter()
            .enumerate()
            .map(|(tag, &key)| Rec { key, tag })
            .collect()
    };
    let cases: [(&str, Vec<Rec>, usize, usize); 6] = [
        ("empty", recs(&[]), 0, 0),
        ("one element", recs(&[7]), 0, 0),
        ("all equal, n=10", recs(&[5; 10]), 9, 0),
        ("sorted, n=10", recs(&[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]), 9, 0),
        (
            "reversed, n=10",
            recs(&[9, 8, 7, 6, 5, 4, 3, 2, 1, 0]),
            45,
            45,
        ),
        ("1, 0, 1 (a tie)", recs(&[1, 0, 1]), 2, 1),
    ];
    println!();
    println!("edge cases, exact expected counts (want = derived by hand, not by running it):");
    println!("  {:<20} {:>12} {:>12}  ok", "input", "want c/s", "got c/s");
    for (name, input, comps, shf) in cases {
        let mut got = input.clone();
        let c = insertion_sort_counted(&mut got, by_key);
        let mut want = input;
        want.sort_by(by_key);
        let ok = c.comparisons == comps && c.shifts == shf && got == want;
        println!(
            "  {name:<20} {:>12} {:>12}  {ok}",
            format!("{comps}/{shf}"),
            format!("{}/{}", c.comparisons, c.shifts)
        );
    }
}

// ------------------------------------------------------------- MEASURED --

// A counting global allocator: Rust's stand-in for testing.AllocsPerRun.
struct Counting;
static ALLOCS: AtomicUsize = AtomicUsize::new(0);
static BYTES: AtomicUsize = AtomicUsize::new(0);

unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, l: Layout) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        BYTES.fetch_add(l.size(), Relaxed);
        unsafe { System.alloc(l) }
    }
    unsafe fn dealloc(&self, p: *mut u8, l: Layout) {
        unsafe { System.dealloc(p, l) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// times each f and returns ns per call of each: best of 12 rounds, where a
/// round gives every f in turn a batch that doubles until it runs >= 5 ms.
/// Interleaving spreads a burst of machine noise over all columns instead of one.
fn ns_per_op(fs: &mut [Box<dyn FnMut(usize) + '_>]) -> Vec<f64> {
    let mut best = vec![f64::MAX; fs.len()];
    for _ in 0..12 {
        for (i, f) in fs.iter_mut().enumerate() {
            let mut batch = 1;
            loop {
                let start = Instant::now();
                for k in 0..batch {
                    f(k);
                }
                let el = start.elapsed();
                if el >= Duration::from_millis(5) {
                    best[i] = best[i].min(el.as_nanos() as f64 / batch as f64);
                    break;
                }
                batch *= 2;
            }
        }
    }
    best
}

/// One timed column: iteration k copies pristine input k into this column's own
/// buffer, then sorts it. (Each closure owns its buffer: five closures cannot
/// share one `&mut Vec`, which is exactly what the borrow checker forbids.)
fn column<'a>(
    pool: &'a [Vec<i64>],
    mut sort: impl FnMut(&mut [i64]) + 'a,
) -> Box<dyn FnMut(usize) + 'a> {
    let mut w = vec![0i64; pool[0].len()];
    Box::new(move |k| {
        w.copy_from_slice(&pool[k % pool.len()]);
        sort(&mut w);
        black_box(&w);
    })
}

/// own sort behind dynamic dispatch: every comparison is an indirect call
/// through a vtable -- what Go's slices.SortFunc always pays. Rust's own sorts
/// and both std sorts are generic over the closure type, so each gets a copy
/// with the comparator inlined (monomorphisation) -- no inlining luck needed.
#[inline(never)]
fn own_dyn(v: &mut [i64], f: &dyn Fn(&i64, &i64) -> Ordering) {
    insertion_sort_by(v, f);
}

type SortFn<'a> = &'a dyn Fn(&mut [i64]);

const NAMES: [&str; 5] = ["own", "own swap", "own dyn", "unstable", "sort_by"];

fn measure() {
    let mut rng = Pcg::new(16, 18);
    let icmp = |a: &i64, b: &i64| a.cmp(b);

    println!();
    println!("MEASURED  ns per sort of Vec<i64> with a.cmp(b) (best of 12 rounds; every");
    println!("iteration copies a pristine input first, and 'copy' times that copy alone)");
    println!("own = rotate_right(1) spelling, own swap = adjacent swaps, own dyn = own through");
    println!("&dyn Fn (not inlined), unstable = sort_unstable_by");
    println!();
    print!("  {:<18} {:>7}", "input", "copy");
    for name in NAMES {
        print!(" {name:>9}");
    }
    println!("  fastest");
    // (column header for the ratio table, insertion sort in std too?, times)
    let mut rows: Vec<(String, bool, Vec<f64>)> = Vec::new();
    let mut row = |label: &str, short: String, pool: &[Vec<i64>]| {
        let mut fs = vec![
            column(pool, |_| {}),
            column(pool, |w| insertion_sort_by(w, icmp)),
            column(pool, |w| insertion_sort_by_swaps(w, icmp)),
            column(pool, |w| own_dyn(w, black_box(&icmp))),
            column(pool, |w| w.sort_unstable_by(icmp)),
            column(pool, |w| w.sort_by(icmp)),
        ];
        let t = ns_per_op(&mut fs);
        let best = (1..t.len()).fold(1, |b, i| if t[i] < t[b] { i } else { b });
        print!("  {label:<18} {:>7.0}", t[0]);
        for x in &t[1..] {
            print!(" {x:>9.0}");
        }
        println!("  {}", NAMES[best - 1]);
        rows.push((short, pool[0].len() <= 20, t));
    };
    // small n: a pool of 1000 different permutations, so the branch predictor
    // cannot memorise one input (Fisher-Yates with int_n: Go's exact pools)
    for n in [4usize, 8, 12, 16, 20, 32, 64] {
        let pool: Vec<Vec<i64>> = (0..1000)
            .map(|_| {
                let mut p: Vec<i64> = (0..n as i64).collect();
                for v in (1..n).rev() {
                    p.swap(v, rng.int_n(v + 1));
                }
                p
            })
            .collect();
        row(&format!("random, n={n}"), format!("n={n}"), &pool);
    }

    // large n, nearly sorted: shuffle each block of k+1 = 5 consecutive values,
    // so every element is at most k = 4 places from home
    const BIG: usize = 100_000;
    const K: usize = 4;
    let mut ks: Vec<i64> = (0..BIG as i64).collect();
    for blk in ks.as_chunks_mut::<{ K + 1 }>().0 {
        for i in (1..=K).rev() {
            blk.swap(i, rng.int_n(i + 1));
        }
    }
    row(
        "4-sorted, n=100000",
        "4-sorted".to_string(),
        std::slice::from_ref(&ks),
    );

    // read the table: computed from this run's numbers, not asserted. Up to
    // len 20 BOTH std sorts run insertion sort (shift version, comparator
    // inlined) -- the same algorithm as own and own swap, so in those rows the
    // ratios compare spellings, not algorithms.
    println!();
    print!("  {:<10}", "time / own");
    for (short, _, _) in &rows {
        print!(" {short:>8}");
    }
    println!();
    let mut geo = [0.0f64; 6];
    for c in 2..6 {
        print!("  {:<10}", NAMES[c - 1]);
        for (_, small, t) in &rows {
            print!(" {:>8.2}", t[c] / t[1]);
            if *small {
                geo[c] += (t[c] / t[1]).ln() / 5.0; // 5 rows with n <= 20
            }
        }
        println!();
    }
    println!(
        "  geometric mean over n = 4..20, where std runs insertion sort too: own swap {:.2},",
        geo[2].exp()
    );
    println!(
        "  own dyn {:.2}, unstable {:.2}, sort_by {:.2} of own's time",
        geo[3].exp(),
        geo[4].exp(),
        geo[5].exp()
    );

    // what each sort pays on that input, in comparator calls
    let mut work = ks.clone();
    let c = insertion_sort_counted(&mut work, icmp);
    let mut calls = [0usize; 2];
    work.copy_from_slice(&ks);
    work.sort_unstable_by(|a, b| {
        calls[0] += 1;
        a.cmp(b)
    });
    work.copy_from_slice(&ks);
    work.sort_by(|a, b| {
        calls[1] += 1;
        a.cmp(b)
    });
    println!(
        "  on the 4-sorted input own made {} shifts (= I) and {} comparisons;",
        c.shifts, c.comparisons
    );
    println!(
        "  comparator calls there: sort_unstable_by {}, sort_by {}",
        calls[0], calls[1]
    );

    // allocations: the claim "Theta(1) space, no allocations", checked
    let mut report = String::new();
    let sorts: [SortFn; 5] = [
        &|w| insertion_sort_by(w, icmp),
        &|w| insertion_sort_by_swaps(w, icmp),
        &|w| own_dyn(w, &icmp),
        &|w| w.sort_unstable_by(icmp),
        &|w| w.sort_by(icmp),
    ];
    for (name, sort) in NAMES.iter().zip(sorts) {
        work.copy_from_slice(&ks);
        let (a0, b0) = (ALLOCS.load(Relaxed), BYTES.load(Relaxed));
        sort(&mut work);
        let (a, b) = (ALLOCS.load(Relaxed) - a0, BYTES.load(Relaxed) - b0);
        let _ = write!(report, " {name} {a}");
        if b > 0 {
            let _ = write!(report, " ({b} bytes)");
        }
    }
    println!("  allocations per sort of it (counting allocator):{report}");
}

fn main() {
    prove();
    edge_cases();
    measure();
    println!();
    println!("from the source: at len <= 20 sort_by and sort_unstable_by both run");
    println!("insertion_sort_shift_left, which shifts through a hole with unsafe code -- so up");
    println!("to there the rows compare own's two safe spellings with std's hole, not algorithms.");
    println!(
        "the own sort is Theta(n + I): it wins where I is small, tiny or nearly sorted input."
    );
    println!("On large random input I ~ n^2/4, and lesson 01 example 11 timed where it loses.");
}
```

**Rust sample output:**

```
PROVED  100000 PCG cases, n in [0, 40], keys in [0, 4), tag = input index
  (a) insertion_sort_by      == sort_by (stable), whole records:  0 mismatches
      insertion_sort_counted == sort_by (stable), whole records:  0 mismatches
      insertion_sort_by_swaps == sort_by (stable), whole records: 0 mismatches
  (b) shifts == brute-force inversions I:                         0 mismatches
  (c) I <= comparisons <= I + n - 1:                              0 violations
  (d) invariant after each of 18591 outer steps (first 1000 cases):  0 violations
      (prefix == stable sort of the original prefix, suffix untouched)
  cases containing equal keys: 92046 of 100000, so stability was exercised, not assumed
  totals: 11525644 comparisons, 9724266 shifts (= the total of I)

  does the harness have teeth? the mutant with != Greater instead of == Less:
    sorted by key:          100000 of 100000   <- a sortedness test passes it
    equal to the oracle:    7954 of 100000   <- the stability check fails it
    smallest failing input (key, tag): [{3 0} {3 1}] -> [{3 1} {3 0}]

edge cases, exact expected counts (want = derived by hand, not by running it):
  input                    want c/s      got c/s  ok
  empty                         0/0          0/0  true
  one element                   0/0          0/0  true
  all equal, n=10               9/0          9/0  true
  sorted, n=10                  9/0          9/0  true
  reversed, n=10              45/45        45/45  true
  1, 0, 1 (a tie)               2/1          2/1  true

MEASURED  ns per sort of Vec<i64> with a.cmp(b) (best of 12 rounds; every
iteration copies a pristine input first, and 'copy' times that copy alone)
own = rotate_right(1) spelling, own swap = adjacent swaps, own dyn = own through
&dyn Fn (not inlined), unstable = sort_unstable_by

  input                 copy       own  own swap   own dyn  unstable   sort_by  fastest
  random, n=4              3        11         6        17         7         6  own swap
  random, n=8              2        29        12        44        13        13  own swap
  random, n=12             2        51        23        86        23        23  own swap
  random, n=16             3        86        41       164        39        41  unstable
  random, n=20             3       124        56       227        53        52  sort_by
  random, n=32             4       267       171       480        76        95  unstable
  random, n=64             7       843       596      1478       178       366  unstable
  4-sorted, n=100000   10967    431727    365203    628182    711693    737536  own swap

  time / own      n=4      n=8     n=12     n=16     n=20     n=32     n=64 4-sorted
  own swap       0.56     0.43     0.44     0.47     0.45     0.64     0.71     0.85
  own dyn        1.53     1.52     1.69     1.92     1.82     1.80     1.75     1.46
  unstable       0.61     0.45     0.45     0.46     0.43     0.29     0.21     1.65
  sort_by        0.58     0.45     0.44     0.48     0.42     0.36     0.43     1.71
  geometric mean over n = 4..20, where std runs insertion sort too: own swap 0.47,
  own dyn 1.69, unstable 0.47, sort_by 0.47 of own's time
  on the 4-sorted input own made 100419 shifts (= I) and 200417 comparisons;
  comparator calls there: sort_unstable_by 1700089, sort_by 1579450
  allocations per sort of it (counting allocator): own 0 own swap 0 own dyn 0 unstable 0 sort_by 1 (800000 bytes)

from the source: at len <= 20 sort_by and sort_unstable_by both run
insertion_sort_shift_left, which shifts through a hole with unsafe code -- so up
to there the rows compare own's two safe spellings with std's hole, not algorithms.
the own sort is Theta(n + I): it wins where I is small, tiny or nearly sorted input.
On large random input I ~ n^2/4, and lesson 01 example 11 timed where it loses.
```

> **Go vs Rust:** The PROVED and edge-case sections match Go's line for line: the same PCG cases, 92,046 with ties, 7,954 mutant passes, the same totals. One extra line proves the swap spelling. Safe Rust cannot leave a hole in a slice, and that changes the timing story. The `rotate_right(1)` spelling took about twice std's time at n ≤ 20, where std runs the same insertion sort (geometric mean: std ≈ 0.47–0.48 of its time). The adjacent-swap spelling matched std there. A plausible reason, read in the source and not measured: `rotate_right` makes a memmove call per insertion.
> Rust needs no `noinline` column, because own and both std sorts are monomorphised with the closure inlined. Forcing an indirect call through `&dyn Fn` cost ≈ 1.4–2.0×. On the 4-sorted input both std sorts made ≈ 1.6–1.7 M comparator calls (Go's `SortStableFunc` made 214,209) and took ≈ 1.6–1.7× own's time. `sort_by` was the only sort that allocated: once, 800,000 bytes (n × 8).

**Complexity:** Θ(n + I) time, Θ(1) space, 0 allocations (`testing.AllocsPerRun`), stable. Proved over 100,000 cases: 92,046 of them contain ties, and in exactly the other 7,954 the `<=` mutant still matched the oracle, though it sorted every case. In every run the inlined own sort was the fastest column in every row, and inlining alone was worth up to ≈ 3.5–3.9×. Behind `//go:noinline` it was level with `SortFunc` where both run insertion sort (n ≤ 12: geometric-mean ratio 0.98–1.12 across runs, so shift vs swap is lost in the noise), and lost to it at n = 64 (`SortFunc` took ≈ 0.84–0.86× its time). On the 4-sorted n = 100,000 input own made 200,417 comparisons against `SortFunc`'s 1,571,768 and was ≈ 4× faster; `SortStableFunc` (214,209 calls) took ≈ 1.1× its time. Do not use it on large random input, where I ≈ n²/4.

---

> Next lesson: [13 — Efficient Sorts: Divide & Conquer](../../13-efficient-sorts.md) — merge sort and
> quicksort, the two ways past the adjacent-swap barrier, and what `slices.Sort` does above n = 12.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
