# Step 15 — Binary Search & Its Variants · 🟡 Medium

Examples **7–11**: the rotated array and the input that breaks it, binary search on the answer (Koko's
bananas, ship packing), the two kinds of sorted matrix, and peak finding in unsorted data.

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

## 7. Searching a rotated array

`🟡 medium` · *Rotated sorted array*

A sorted array rotated by k is two sorted runs, and binary search still works on it because one
comparison tells you which side of `mid` the drop is on: `a[mid] > a[hi]` means the minimum lies to
the right. The same idea searches for a target in one pass. At least one half of the window is
sorted, so you check whether the target falls inside that half's value range. Both are Θ(log n),
and the program checks every rotation of every n ≤ 12 against a linear scan. Then it breaks the
trick: once duplicates are allowed, `a[mid] == a[hi]` tells you nothing, the only safe move is
`hi--`, and on an array of 1s with a single 0 the probe count climbs to n − 1. That is Θ(n).

**Steps:**

1. Write `findMin` (compare `a[mid]` with `a[hi]`) and the one-pass `search`, and trace both on a
   rotation of 8 values.
2. Check every n from 1 to 12, every rotation and every target, present or absent, against a linear
   scan, and record the worst probe count next to its bound.
3. Put a single 0 among 999 ones at every position: count the probes the duplicate-safe version
   needs, and count how often the distinct-values version returns the wrong answer.

```go
package main

import (
	"fmt"
	"math/bits"
)

// A sorted array rotated by k is two sorted runs glued together:
//
//	sorted   [ 2  5  9 12 15 18 22 30]
//	rotated  [15 18 22 30  2  5  9 12]   (k = 4: the first 4 moved to the back)
//
// Binary search still works, because ONE comparison tells you which side of
// mid the "drop" (the rotation point) is on.

// findMin returns the index of the minimum (the rotation point). The window is
// [lo, hi] with hi = the last index, because we compare a[mid] with a[hi]:
//
//	a[mid] > a[hi]  -> the drop is right of mid: the minimum is in (mid, hi]
//	a[mid] < a[hi]  -> mid..hi is one sorted run: the minimum is in [lo, mid]
//
// Correct only for DISTINCT values: a[mid] == a[hi] cannot happen when mid < hi.
func findMin(a []int, probes *int) int {
	lo, hi := 0, len(a)-1
	for lo < hi {
		mid := lo + (hi-lo)/2
		*probes++
		if a[mid] > a[hi] {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo
}

// search finds target in one pass. Half-open window [lo, hi), like example 1.
// At least one of the halves [lo, mid] and [mid, hi) is sorted; if target lies
// inside the sorted one's value range, go there, otherwise go to the other one.
func search(a []int, target int, probes *int, trace bool) int {
	lo, hi := 0, len(a)
	for lo < hi {
		mid := lo + (hi-lo)/2
		*probes++
		if trace {
			fmt.Printf("    [%d,%d) mid=%d a[mid]=%2d  ", lo, hi, mid, a[mid])
		}
		if a[mid] == target {
			if trace {
				fmt.Println("== target")
			}
			return mid
		}
		var half, move string
		if a[lo] <= a[mid] { // left part [lo, mid] is sorted
			half = "left sorted "
			if a[lo] <= target && target < a[mid] {
				hi, move = mid, "target in it -> go left"
			} else {
				lo, move = mid+1, "target not in it -> go right"
			}
		} else { // right part [mid, hi) is sorted
			half = "right sorted"
			if a[mid] < target && target <= a[hi-1] {
				lo, move = mid+1, "target in it -> go right"
			} else {
				hi, move = mid, "target not in it -> go left"
			}
		}
		if trace {
			fmt.Printf("%s, %s\n", half, move)
		}
	}
	return -1
}

// findMinDup tolerates duplicates. When a[mid] == a[hi] we learn NOTHING about
// which side holds the minimum, so the only safe move is to drop a[hi] (it has
// a twin at mid, so the minimum value survives). That step shrinks the window
// by ONE, not by half: the Theta(log n) argument is gone.
func findMinDup(a []int, probes *int) int {
	lo, hi := 0, len(a)-1
	for lo < hi {
		mid := lo + (hi-lo)/2
		*probes++
		switch {
		case a[mid] > a[hi]:
			lo = mid + 1
		case a[mid] < a[hi]:
			hi = mid
		default:
			hi--
		}
	}
	return lo
}

func rotate(sorted []int, k int) []int {
	out := make([]int, 0, len(sorted))
	out = append(out, sorted[k:]...)
	return append(out, sorted[:k]...)
}

func linearFind(a []int, target int) int {
	for i, v := range a {
		if v == target {
			return i
		}
	}
	return -1
}

func main() {
	sorted := []int{2, 5, 9, 12, 15, 18, 22, 30}
	a := rotate(sorted, 4)
	fmt.Println("(a) a =", a, " (sorted, rotated left by 4)")
	p := 0
	m := findMin(a, &p)
	fmt.Printf("  findMin -> index %d (value %d) in %d probes\n", m, a[m], p)
	for _, t := range []int{5, 22, 13} {
		fmt.Printf("  search(%d):\n", t)
		p = 0
		i := search(a, t, &p, true)
		fmt.Printf("    -> %d after %d probes\n", i, p)
	}

	// (b) Every n, every rotation, every target (present values are 0,2,4,..;
	// the odd ones and -1, 2n are absent), checked against a linear scan.
	fmt.Println()
	fmt.Println("(b) exhaustive check against a linear scan, distinct values 0,2,4,...")
	fmt.Println("  probes = loop iterations (one read of a[mid] each), worst case over all rotations and targets")
	fmt.Printf("  %3s %6s %6s %11s %12s %12s %13s\n", "n", "cases", "wrong", "search max", "bits.Len(n)", "findMin max", "ceil(log2 n)")
	total, wrong, overBound := 0, 0, 0
	for n := 1; n <= 12; n++ {
		s := make([]int, n)
		for i := range s {
			s[i] = 2 * i
		}
		cases, bad, maxS, maxM := 0, 0, 0, 0
		for k := 0; k < n; k++ {
			r := rotate(s, k)
			pm := 0
			m := findMin(r, &pm)
			cases++
			if r[m] != 0 { // the minimum value is 0
				bad++
			}
			maxM = max(maxM, pm)
			for t := -1; t <= 2*n; t++ {
				ps := 0
				cases++
				if search(r, t, &ps, false) != linearFind(r, t) {
					bad++
				}
				maxS = max(maxS, ps)
			}
		}
		total += cases
		wrong += bad
		ceilLog := bits.Len(uint(n - 1)) // ceil(log2 n) for n >= 1
		if maxS > bits.Len(uint(n)) || maxM > ceilLog {
			overBound++
		}
		fmt.Printf("  %3d %6d %6d %11d %12d %12d %13d\n", n, cases, bad, maxS, bits.Len(uint(n)), maxM, ceilLog)
	}
	fmt.Printf("  total %d cases (findMin + search), %d wrong\n", total, wrong)
	fmt.Printf("  rows where a worst case exceeded its bound: %d\n", overBound)

	// (c) Duplicates: 999 ones and a single 0, at every position.
	const n = 1000
	fmt.Println()
	fmt.Printf("(c) duplicates: %d ones with a single 0 placed at every position p\n", n-1)
	minP, maxP, sumP, worstPos, wrongDistinct, firstWrong := n+1, 0, 0, 0, 0, -1
	for pos := 0; pos < n; pos++ {
		d := make([]int, n)
		for i := range d {
			d[i] = 1
		}
		d[pos] = 0
		pd := 0
		if got := findMinDup(d, &pd); d[got] != 0 {
			fmt.Println("  findMinDup WRONG at p =", pos) // never printed if the code is right
		}
		sumP += pd
		minP = min(minP, pd)
		if pd > maxP {
			maxP, worstPos = pd, pos
		}
		if got := findMin(d, new(int)); d[got] != 0 {
			wrongDistinct++
			if firstWrong < 0 {
				firstWrong = pos
			}
		}
	}
	fmt.Printf("  findMinDup probes: min %d, avg %.1f, max %d (at p = %d)\n", minP, float64(sumP)/n, maxP, worstPos)
	fmt.Printf("  with distinct values findMin's bound is ceil(log2 %d) = %d probes; the worst case here is %.0fx that\n", n, bits.Len(uint(n-1)), float64(maxP)/float64(bits.Len(uint(n-1))))
	fmt.Printf("  distinct-values findMin on the same inputs: wrong for %d of %d positions\n", wrongDistinct, n)
	if firstWrong >= 0 {
		d := make([]int, n)
		for i := range d {
			d[i] = 1
		}
		d[firstWrong] = 0
		got := findMin(d, new(int))
		fmt.Printf("  e.g. p = %d: it returns index %d, holding %d -- a[mid] == a[hi] sent it the wrong way\n", firstWrong, got, d[got])
	}
}
```

**Output:**

```
(a) a = [15 18 22 30 2 5 9 12]  (sorted, rotated left by 4)
  findMin -> index 4 (value 2) in 3 probes
  search(5):
    [0,8) mid=4 a[mid]= 2  right sorted, target in it -> go right
    [5,8) mid=6 a[mid]= 9  left sorted , target in it -> go left
    [5,6) mid=5 a[mid]= 5  == target
    -> 5 after 3 probes
  search(22):
    [0,8) mid=4 a[mid]= 2  right sorted, target not in it -> go left
    [0,4) mid=2 a[mid]=22  == target
    -> 2 after 2 probes
  search(13):
    [0,8) mid=4 a[mid]= 2  right sorted, target not in it -> go left
    [0,4) mid=2 a[mid]=22  left sorted , target not in it -> go right
    [3,4) mid=3 a[mid]=30  left sorted , target not in it -> go right
    -> -1 after 3 probes

(b) exhaustive check against a linear scan, distinct values 0,2,4,...
  probes = loop iterations (one read of a[mid] each), worst case over all rotations and targets
    n  cases  wrong  search max  bits.Len(n)  findMin max  ceil(log2 n)
    1      5      0           1            1            0             0
    2     14      0           2            2            1             1
    3     27      0           2            2            2             2
    4     44      0           3            3            2             2
    5     65      0           3            3            3             3
    6     90      0           3            3            3             3
    7    119      0           3            3            3             3
    8    152      0           4            4            3             3
    9    189      0           4            4            4             4
   10    230      0           4            4            4             4
   11    275      0           4            4            4             4
   12    324      0           4            4            4             4
  total 1534 cases (findMin + search), 0 wrong
  rows where a worst case exceeded its bound: 0

(c) duplicates: 999 ones with a single 0 placed at every position p
  findMinDup probes: min 9, avg 382.7, max 999 (at p = 0)
  with distinct values findMin's bound is ceil(log2 1000) = 10 probes; the worst case here is 100x that
  distinct-values findMin on the same inputs: wrong for 989 of 1000 positions
  e.g. p = 2: it returns index 0, holding 1 -- a[mid] == a[hi] sent it the wrong way
```

**Rust:**

```rust
// A sorted array rotated by k is two sorted runs glued together:
//
//   sorted   [ 2  5  9 12 15 18 22 30]
//   rotated  [15 18 22 30  2  5  9 12]   (k = 4: rotate_left(4))
//
// Binary search still works, because ONE comparison tells you which side of
// mid the "drop" (the rotation point) is on.

/// Index of the minimum (the rotation point), `None` for an empty slice.
/// The window is [lo, hi] with hi = the last index, because we compare
/// a[mid] with a[hi]:
///   a[mid] > a[hi]  -> the drop is right of mid: the minimum is in (mid, hi]
///   a[mid] < a[hi]  -> mid..hi is one sorted run: the minimum is in [lo, mid]
/// Correct only for DISTINCT values.
fn find_min(a: &[i64], probes: &mut u32) -> Option<usize> {
    // Go writes hi := len(a)-1 and gets -1 on an empty slice; with usize that
    // would underflow, so checked_sub turns "empty" into None up front.
    let (mut lo, mut hi) = (0, a.len().checked_sub(1)?);
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        *probes += 1;
        if a[mid] > a[hi] {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    Some(lo)
}

/// One-pass search, half-open window [lo, hi) like example 1. At least one of
/// [lo, mid] and [mid, hi) is sorted; if target lies inside the sorted one's
/// value range go there, otherwise go to the other one. The half-open form
/// also means we never write `hi = mid - 1`, which underflows usize at mid = 0.
fn search(a: &[i64], target: i64, probes: &mut u32, trace: bool) -> Option<usize> {
    let (mut lo, mut hi) = (0, a.len());
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        *probes += 1;
        if trace {
            print!("    [{lo},{hi}) mid={mid} a[mid]={:>2}  ", a[mid]);
        }
        if a[mid] == target {
            if trace {
                println!("== target");
            }
            return Some(mid);
        }
        let (half, step);
        if a[lo] <= a[mid] {
            // left part [lo, mid] is sorted
            half = "left sorted ";
            if (a[lo]..a[mid]).contains(&target) {
                (hi, step) = (mid, "target in it -> go left");
            } else {
                (lo, step) = (mid + 1, "target not in it -> go right");
            }
        } else {
            // right part [mid, hi) is sorted
            half = "right sorted";
            if a[mid] < target && target <= a[hi - 1] {
                (lo, step) = (mid + 1, "target in it -> go right");
            } else {
                (hi, step) = (mid, "target not in it -> go left");
            }
        }
        if trace {
            println!("{half}, {step}");
        }
    }
    None
}

/// Tolerates duplicates. When a[mid] == a[hi] we learn NOTHING about which side
/// holds the minimum, so the only safe move is to drop a[hi] (its twin at mid
/// keeps the minimum value alive). That shrinks the window by ONE, not by half:
/// the Theta(log n) argument is gone. (`hi -= 1` is safe: hi > lo >= 0.)
fn find_min_dup(a: &[i64], probes: &mut u32) -> Option<usize> {
    let (mut lo, mut hi) = (0, a.len().checked_sub(1)?);
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        *probes += 1;
        match a[mid].cmp(&a[hi]) {
            std::cmp::Ordering::Greater => lo = mid + 1,
            std::cmp::Ordering::Less => hi = mid,
            std::cmp::Ordering::Equal => hi -= 1,
        }
    }
    Some(lo)
}

fn rotated(sorted: &[i64], k: usize) -> Vec<i64> {
    let mut v = sorted.to_vec();
    v.rotate_left(k); // in place, Theta(n); Go has no slices.Rotate
    v
}

/// Go prints slices as [1 2 3]; Rust's {:?} would print [1, 2, 3].
fn go_slice(a: &[i64]) -> String {
    let parts: Vec<String> = a.iter().map(i64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

/// Go's bits.Len(x): the number of bits needed to write x.
fn bits_len(x: usize) -> u32 {
    usize::BITS - x.leading_zeros()
}

fn main() {
    let sorted = [2, 5, 9, 12, 15, 18, 22, 30];
    let a = rotated(&sorted, 4);
    println!("(a) a = {}  (sorted, rotated left by 4)", go_slice(&a));
    let mut p = 0;
    let m = find_min(&a, &mut p).expect("non-empty");
    println!("  findMin -> index {m} (value {}) in {p} probes", a[m]);
    for t in [5, 22, 13] {
        println!("  search({t}):");
        p = 0;
        // Option instead of Go's -1; printed as -1 so the two outputs line up
        let shown = search(&a, t, &mut p, true).map_or(-1, |i| i as i64);
        println!("    -> {shown} after {p} probes");
    }

    // (b) Every n, every rotation, every target (present values are 0,2,4,..;
    // the odd ones and -1, 2n are absent), checked against a linear scan.
    println!();
    println!("(b) exhaustive check against a linear scan, distinct values 0,2,4,...");
    println!(
        "  probes = loop iterations (one read of a[mid] each), worst case over all rotations and targets"
    );
    println!(
        "  {:>3} {:>6} {:>6} {:>11} {:>12} {:>12} {:>13}",
        "n", "cases", "wrong", "search max", "bits.Len(n)", "findMin max", "ceil(log2 n)"
    );
    let (mut total, mut wrong, mut over_bound) = (0, 0, 0);
    for n in 1..=12usize {
        let s: Vec<i64> = (0..n as i64).map(|i| 2 * i).collect();
        let (mut cases, mut bad, mut max_s, mut max_m) = (0, 0, 0, 0);
        for k in 0..n {
            let r = rotated(&s, k);
            let mut pm = 0;
            let m = find_min(&r, &mut pm).expect("n >= 1");
            cases += 1;
            if r[m] != 0 {
                bad += 1;
            }
            max_m = max_m.max(pm);
            for t in -1..=2 * n as i64 {
                let mut ps = 0;
                cases += 1;
                // the oracle: position() is the linear scan
                if search(&r, t, &mut ps, false) != r.iter().position(|&v| v == t) {
                    bad += 1;
                }
                max_s = max_s.max(ps);
            }
        }
        total += cases;
        wrong += bad;
        let ceil_log = bits_len(n - 1); // ceil(log2 n) for n >= 1
        if max_s > bits_len(n) || max_m > ceil_log {
            over_bound += 1;
        }
        println!(
            "  {n:>3} {cases:>6} {bad:>6} {max_s:>11} {:>12} {max_m:>12} {ceil_log:>13}",
            bits_len(n)
        );
    }
    println!("  total {total} cases (findMin + search), {wrong} wrong");
    println!("  rows where a worst case exceeded its bound: {over_bound}");

    // (c) Duplicates: 999 ones and a single 0, at every position.
    const N: usize = 1000;
    println!();
    println!(
        "(c) duplicates: {} ones with a single 0 placed at every position p",
        N - 1
    );
    let (mut min_p, mut max_p, mut sum_p, mut worst_pos) = (u32::MAX, 0, 0, 0);
    let (mut wrong_distinct, mut first_wrong) = (0, None);
    for pos in 0..N {
        let mut d = vec![1i64; N];
        d[pos] = 0;
        let mut pd = 0;
        let got = find_min_dup(&d, &mut pd).expect("non-empty");
        if d[got] != 0 {
            println!("  findMinDup WRONG at p = {pos}"); // never printed if the code is right
        }
        sum_p += pd;
        min_p = min_p.min(pd);
        if pd > max_p {
            (max_p, worst_pos) = (pd, pos);
        }
        let got = find_min(&d, &mut 0).expect("non-empty");
        if d[got] != 0 {
            wrong_distinct += 1;
            first_wrong.get_or_insert(pos);
        }
    }
    println!(
        "  findMinDup probes: min {min_p}, avg {:.1}, max {max_p} (at p = {worst_pos})",
        f64::from(sum_p) / N as f64
    );
    let bound = bits_len(N - 1);
    println!(
        "  with distinct values findMin's bound is ceil(log2 {N}) = {bound} probes; the worst case here is {:.0}x that",
        f64::from(max_p) / f64::from(bound)
    );
    println!(
        "  distinct-values findMin on the same inputs: wrong for {wrong_distinct} of {N} positions"
    );
    if let Some(pos) = first_wrong {
        let mut d = vec![1i64; N];
        d[pos] = 0;
        let got = find_min(&d, &mut 0).expect("non-empty");
        println!(
            "  e.g. p = {pos}: it returns index {got}, holding {} -- a[mid] == a[hi] sent it the wrong way",
            d[got]
        );
    }
}
```

**Rust output:**

```
(a) a = [15 18 22 30 2 5 9 12]  (sorted, rotated left by 4)
  findMin -> index 4 (value 2) in 3 probes
  search(5):
    [0,8) mid=4 a[mid]= 2  right sorted, target in it -> go right
    [5,8) mid=6 a[mid]= 9  left sorted , target in it -> go left
    [5,6) mid=5 a[mid]= 5  == target
    -> 5 after 3 probes
  search(22):
    [0,8) mid=4 a[mid]= 2  right sorted, target not in it -> go left
    [0,4) mid=2 a[mid]=22  == target
    -> 2 after 2 probes
  search(13):
    [0,8) mid=4 a[mid]= 2  right sorted, target not in it -> go left
    [0,4) mid=2 a[mid]=22  left sorted , target not in it -> go right
    [3,4) mid=3 a[mid]=30  left sorted , target not in it -> go right
    -> -1 after 3 probes

(b) exhaustive check against a linear scan, distinct values 0,2,4,...
  probes = loop iterations (one read of a[mid] each), worst case over all rotations and targets
    n  cases  wrong  search max  bits.Len(n)  findMin max  ceil(log2 n)
    1      5      0           1            1            0             0
    2     14      0           2            2            1             1
    3     27      0           2            2            2             2
    4     44      0           3            3            2             2
    5     65      0           3            3            3             3
    6     90      0           3            3            3             3
    7    119      0           3            3            3             3
    8    152      0           4            4            3             3
    9    189      0           4            4            4             4
   10    230      0           4            4            4             4
   11    275      0           4            4            4             4
   12    324      0           4            4            4             4
  total 1534 cases (findMin + search), 0 wrong
  rows where a worst case exceeded its bound: 0

(c) duplicates: 999 ones with a single 0 placed at every position p
  findMinDup probes: min 9, avg 382.7, max 999 (at p = 0)
  with distinct values findMin's bound is ceil(log2 1000) = 10 probes; the worst case here is 100x that
  distinct-values findMin on the same inputs: wrong for 989 of 1000 positions
  e.g. p = 2: it returns index 0, holding 1 -- a[mid] == a[hi] sent it the wrong way
```

> **Go vs Rust:** Identical output (diffed): same probes, same 1,534 cases, same 999-probe worst case. Rust's `usize` changes two lines. Go's `hi := len(a)-1` becomes `a.len().checked_sub(1)?`, so an empty slice returns `None` instead of underflowing. `search` uses the half-open window so it never writes `hi = mid - 1`, which would underflow at `mid = 0`. `rotate_left` rotates in place, while Go has no `slices.Rotate` and builds the rotation with two `append`s. `search` returns `Option<usize>` where Go returns −1, and the oracle is `iter().position`.

**Complexity:** Θ(log n) time, Θ(1) space with distinct values. `search` never exceeded `bits.Len(n)` probes and `findMin` never exceeded ⌈log₂ n⌉ over 1,534 exhaustive cases, with 0 wrong. With duplicates the worst case is Θ(n): 999 probes for n = 1000 (average 382.7), and the distinct-values version is wrong for 989 of the 1000 inputs.

---

## 8. Binary search on the answer: Koko's bananas

`🟡 medium` · *Search the answer space*

Here no array is searched. The answer itself, an eating speed k, lies somewhere in
`[1, max pile]`, and `feasible(k) = Σ⌈p/k⌉ ≤ h` is monotone because eating faster never takes
longer. A monotone predicate over a range is all binary search needs. You get Θ(n log max p) instead
of the Θ(n · answer) of trying k = 1, 2, 3, …, and the ceiling is the integer `(p + k − 1) / k`, with
no floats. The program counts feasibility calls on 10,000 piles of up to 10⁹ bananas and checks
2,000 random small cases against brute force. It also checks that the predicate really is monotone,
because a binary search over a non-monotone predicate still returns an answer, just a wrong one.

**Steps:**

1. Write `hours(piles, k)` with the integer ceiling, and `firstTrue(lo, hi, f)`, the lower-bound
   loop of example 1 over a range of values.
2. Search k in `[1, max(piles)]` and trace every probe on the classic case, then run the 10,000-pile
   case and compare its call count with the k calls a scan from 1 would make.
3. Check the answer against brute force on 2,000 PCG cases, and print `feasible(k)` for every k of
   one case to see its single 0→1 boundary.

```go
package main

import (
	"fmt"
	"math/bits"
	"math/rand/v2"
	"slices"
	"strings"
)

// Koko eats bananas at k per hour: each hour she picks one pile and eats k
// from it (or the rest of it, if fewer remain). Pile p therefore takes
// ceil(p/k) hours. What is the smallest k that finishes all piles in h hours?
//
// There is no array to search. The ANSWER k lives in [1, max(piles)] (at
// k = max every pile takes one hour, and h >= len(piles) so that always
// works), and the predicate
//
//	feasible(k) = sum of ceil(p/k) <= h
//
// is monotone: eating faster never takes longer. So it is false...false,
// true...true over k, and the first true is the answer. Binary search finds
// it in log2(max p) calls to feasible; each call costs Theta(n).

// hours is the total time at speed k. Integer ceiling, no floats:
// ceil(p/k) == (p + k - 1) / k for p >= 0, k >= 1.
func hours(piles []int64, k int64) int64 {
	var total int64
	for _, p := range piles {
		total += (p + k - 1) / k
	}
	return total
}

// firstTrue returns the smallest x in [lo, hi] with f(x) true, given that
// f is monotone and f(hi) is true. Same template as example 1, over values:
// the window [lo, hi] always contains the answer, and mid never equals hi,
// so it shrinks every iteration. lo + (hi-lo)/2 avoids the overflow of
// example 2 (it matters for value ranges, not here, but it costs nothing).
func firstTrue(lo, hi int64, f func(int64) bool) int64 {
	for lo < hi {
		mid := lo + (hi-lo)/2
		if f(mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

// minSpeed solves the problem and reports how many times feasible ran.
func minSpeed(piles []int64, h int64, trace bool) (k int64, calls int) {
	feasible := func(k int64) bool {
		calls++
		t := hours(piles, k)
		if trace {
			fmt.Printf("    k = %2d  hours = %2d  feasible = %v\n", k, t, t <= h)
		}
		return t <= h
	}
	return firstTrue(1, slices.Max(piles), feasible), calls
}

// bruteSpeed is the oracle: try k = 1, 2, 3, ... and stop at the first that
// works. Obviously correct, and it makes exactly `answer` calls.
func bruteSpeed(piles []int64, h int64) int64 {
	for k := int64(1); ; k++ {
		if hours(piles, k) <= h {
			return k
		}
	}
}

func main() {
	// (a) The classic case.
	piles := []int64{3, 6, 7, 11}
	var h int64 = 8
	fmt.Printf("(a) piles = %v, h = %d, k searched in [1, %d]\n", piles, h, slices.Max(piles))
	k, calls := minSpeed(piles, h, true)
	fmt.Printf("  answer k = %d after %d feasibility calls\n", k, calls)

	// (d, shown early because it explains (a)) the predicate over every k.
	var sb strings.Builder
	flips, prev := 0, false
	for k := int64(1); k <= slices.Max(piles); k++ {
		ok := hours(piles, k) <= h
		if ok {
			sb.WriteByte('1')
		} else {
			sb.WriteByte('0')
		}
		if k > 1 && ok != prev {
			flips++
		}
		prev = ok
	}
	verdict := "exactly one 0->1 boundary: monotone"
	if flips != 1 {
		verdict = fmt.Sprintf("%d changes: NOT a clean boundary", flips)
	}
	fmt.Printf("  feasible(k) for k = 1..%d: %s  (%s)\n", slices.Max(piles), sb.String(), verdict)

	// (b) A big case: 10,000 piles of up to 10^9 bananas, h = 2n.
	rng := rand.New(rand.NewPCG(1, 2))
	const n = 10_000
	big := make([]int64, n)
	for i := range big {
		big[i] = int64(rng.IntN(1_000_000_000)) + 1
	}
	hb := int64(2 * n)
	maxP := slices.Max(big)
	fmt.Println()
	fmt.Printf("(b) %d PCG piles, max pile %d, h = %d\n", n, maxP, hb)
	k, calls = minSpeed(big, hb, false)
	fmt.Printf("  answer k = %d\n", k)
	fmt.Printf("  hours at k-1 = %d (> h), at k = %d (<= h): k is the boundary\n", hours(big, k-1), hours(big, k))
	fmt.Printf("  binary search: %d feasibility calls  (bits.Len(max pile) = %d)\n", calls, bits.Len64(uint64(maxP)))
	fmt.Printf("  a scan from k = 1 would make %d calls (= the answer; computed, not run)\n", k)
	fmt.Printf("  pile divisions: %d vs %d, about %.0fx fewer\n", int64(calls)*n, k*n, float64(k)/float64(calls))

	// (c) Oracle: 2,000 small random cases against the brute-force scan.
	fmt.Println()
	const trials = 2000
	mismatches, nonMonotone, searchCalls, bruteCalls, maxCalls := 0, 0, 0, int64(0), 0
	for range trials {
		m := rng.IntN(8) + 1
		ps := make([]int64, m)
		for i := range ps {
			ps[i] = int64(rng.IntN(30)) + 1
		}
		hh := int64(m + rng.IntN(2*m+1)) // h in [n, 3n]
		got, c := minSpeed(ps, hh, false)
		want := bruteSpeed(ps, hh)
		if got != want {
			mismatches++
			if mismatches <= 3 {
				fmt.Println("  MISMATCH", ps, hh, got, want)
			}
		}
		// monotonicity, checked rather than assumed: once true, never false again
		seenTrue := false
		for k := int64(1); k <= slices.Max(ps); k++ {
			ok := hours(ps, k) <= hh
			if seenTrue && !ok {
				nonMonotone++
				break
			}
			seenTrue = seenTrue || ok
		}
		searchCalls += c
		bruteCalls += want
		maxCalls = max(maxCalls, c)
	}
	fmt.Printf("(c) %d random cases (1-8 piles of 1-30, h in [n, 3n]) vs brute force: %d mismatches\n", trials, mismatches)
	fmt.Printf("  cases where feasible(k) went true -> false somewhere in [1, max]: %d\n", nonMonotone)
	fmt.Printf("  feasibility calls: binary %d (max %d per case), brute force %d\n", searchCalls, maxCalls, bruteCalls)
}
```

**Output:**

```
(a) piles = [3 6 7 11], h = 8, k searched in [1, 11]
    k =  6  hours =  6  feasible = true
    k =  3  hours = 10  feasible = false
    k =  5  hours =  8  feasible = true
    k =  4  hours =  8  feasible = true
  answer k = 4 after 4 feasibility calls
  feasible(k) for k = 1..11: 00011111111  (exactly one 0->1 boundary: monotone)

(b) 10000 PCG piles, max pile 999978963, h = 20000
  answer k = 333270263
  hours at k-1 = 20001 (> h), at k = 20000 (<= h): k is the boundary
  binary search: 29 feasibility calls  (bits.Len(max pile) = 30)
  a scan from k = 1 would make 333270263 calls (= the answer; computed, not run)
  pile divisions: 290000 vs 3332702630000, about 11492078x fewer

(c) 2000 random cases (1-8 piles of 1-30, h in [n, 3n]) vs brute force: 0 mismatches
  cases where feasible(k) went true -> false somewhere in [1, max]: 0
  feasibility calls: binary 8709 (max 5 per case), brute force 23407
```

**Rust:**

```rust
// Koko eats bananas at k per hour: each hour she picks one pile and eats k
// from it (or the rest of it). Pile p takes ceil(p/k) hours. What is the
// smallest k that finishes all piles in h hours?
//
// There is no array to search. The ANSWER k lives in [1, max(piles)] (at
// k = max every pile takes one hour, and h >= len(piles) so that works), and
//
//   feasible(k) = sum of ceil(p/k) <= h
//
// is monotone: eating faster never takes longer. So it is false...false,
// true...true over k, and the first true is the answer: log2(max p) calls to
// feasible, each Theta(n).

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) gives exactly the piles Go generates.
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

/// Total time at speed k. `div_ceil` is the integer ceiling Go writes as
/// (p + k - 1) / k -- no floats, and no `p + k - 1` to overflow.
fn hours(piles: &[u64], k: u64) -> u64 {
    piles.iter().map(|p| p.div_ceil(k)).sum()
}

/// Smallest x in [lo, hi] with f(x) true, given f monotone and f(hi) true.
/// Rust's std has no binary search over an integer range (`partition_point`
/// needs a slice, and collecting 10^9 candidates is not an option), so this
/// is the 8-line loop of example 1, over values. `u64::midpoint` rounds down
/// and cannot overflow -- the fix from example 2, built in.
fn first_true(mut lo: u64, mut hi: u64, mut f: impl FnMut(u64) -> bool) -> u64 {
    while lo < hi {
        let mid = lo.midpoint(hi);
        if f(mid) {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    lo
}

/// The answer and how many times feasible ran.
fn min_speed(piles: &[u64], h: u64, trace: bool) -> (u64, usize) {
    let max_p = *piles.iter().max().expect("at least one pile");
    let mut calls = 0;
    // the closure borrows `calls` mutably; the borrow ends when first_true returns
    let k = first_true(1, max_p, |k| {
        calls += 1;
        let t = hours(piles, k);
        if trace {
            println!("    k = {k:>2}  hours = {t:>2}  feasible = {}", t <= h);
        }
        t <= h
    });
    (k, calls)
}

/// The oracle: k = 1, 2, 3, ... until one works. Exactly `answer` calls.
fn brute_speed(piles: &[u64], h: u64) -> u64 {
    (1..)
        .find(|&k| hours(piles, k) <= h)
        .expect("k = max pile works")
}

fn go_slice(a: &[u64]) -> String {
    let parts: Vec<String> = a.iter().map(u64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn main() {
    // (a) The classic case.
    let piles = [3u64, 6, 7, 11];
    let h = 8;
    let max_p = *piles.iter().max().expect("non-empty");
    println!(
        "(a) piles = {}, h = {h}, k searched in [1, {max_p}]",
        go_slice(&piles)
    );
    let (k, calls) = min_speed(&piles, h, true);
    println!("  answer k = {k} after {calls} feasibility calls");

    // (d, shown early because it explains (a)) the predicate over every k.
    let bits: Vec<bool> = (1..=max_p).map(|k| hours(&piles, k) <= h).collect();
    let line: String = bits.iter().map(|&b| if b { '1' } else { '0' }).collect();
    let flips = bits.windows(2).filter(|w| w[0] != w[1]).count();
    let verdict = if flips == 1 {
        "exactly one 0->1 boundary: monotone".to_string()
    } else {
        format!("{flips} changes: NOT a clean boundary")
    };
    println!("  feasible(k) for k = 1..{max_p}: {line}  ({verdict})");

    // (b) A big case: 10,000 piles of up to 10^9 bananas, h = 2n.
    let mut rng = Pcg::new(1, 2);
    const N: u64 = 10_000;
    let big: Vec<u64> = (0..N).map(|_| rng.int_n(1_000_000_000) + 1).collect();
    let hb = 2 * N;
    let max_p = *big.iter().max().expect("non-empty");
    println!();
    println!("(b) {N} PCG piles, max pile {max_p}, h = {hb}");
    let (k, calls) = min_speed(&big, hb, false);
    println!("  answer k = {k}");
    println!(
        "  hours at k-1 = {} (> h), at k = {} (<= h): k is the boundary",
        hours(&big, k - 1),
        hours(&big, k)
    );
    println!(
        "  binary search: {calls} feasibility calls  (bits.Len(max pile) = {})",
        u64::BITS - max_p.leading_zeros()
    );
    println!("  a scan from k = 1 would make {k} calls (= the answer; computed, not run)");
    println!(
        "  pile divisions: {} vs {}, about {:.0}x fewer",
        calls as u64 * N,
        k * N,
        k as f64 / calls as f64
    );

    // (c) Oracle: 2,000 small random cases against the brute-force scan.
    println!();
    const TRIALS: usize = 2000;
    let (mut mismatches, mut non_monotone, mut search_calls, mut brute_calls, mut max_calls) =
        (0, 0, 0, 0, 0);
    for _ in 0..TRIALS {
        let m = rng.int_n(8) + 1;
        let ps: Vec<u64> = (0..m).map(|_| rng.int_n(30) + 1).collect();
        let hh = m + rng.int_n(2 * m + 1); // h in [n, 3n]
        let (got, c) = min_speed(&ps, hh, false);
        let want = brute_speed(&ps, hh);
        if got != want {
            mismatches += 1;
            if mismatches <= 3 {
                println!("  MISMATCH {} {hh} {got} {want}", go_slice(&ps));
            }
        }
        // monotonicity, checked rather than assumed: once true, never false again
        let max_k = *ps.iter().max().expect("non-empty");
        let first = (1..=max_k).position(|k| hours(&ps, k) <= hh);
        if let Some(i) = first
            && (i as u64 + 1..=max_k).any(|k| hours(&ps, k) > hh)
        {
            non_monotone += 1;
        }
        search_calls += c;
        brute_calls += want;
        max_calls = max_calls.max(c);
    }
    println!(
        "(c) {TRIALS} random cases (1-8 piles of 1-30, h in [n, 3n]) vs brute force: {mismatches} mismatches"
    );
    println!("  cases where feasible(k) went true -> false somewhere in [1, max]: {non_monotone}");
    println!(
        "  feasibility calls: binary {search_calls} (max {max_calls} per case), brute force {brute_calls}"
    );
}
```

**Rust output:**

```
(a) piles = [3 6 7 11], h = 8, k searched in [1, 11]
    k =  6  hours =  6  feasible = true
    k =  3  hours = 10  feasible = false
    k =  5  hours =  8  feasible = true
    k =  4  hours =  8  feasible = true
  answer k = 4 after 4 feasibility calls
  feasible(k) for k = 1..11: 00011111111  (exactly one 0->1 boundary: monotone)

(b) 10000 PCG piles, max pile 999978963, h = 20000
  answer k = 333270263
  hours at k-1 = 20001 (> h), at k = 20000 (<= h): k is the boundary
  binary search: 29 feasibility calls  (bits.Len(max pile) = 30)
  a scan from k = 1 would make 333270263 calls (= the answer; computed, not run)
  pile divisions: 290000 vs 3332702630000, about 11492078x fewer

(c) 2000 random cases (1-8 piles of 1-30, h in [n, 3n]) vs brute force: 0 mismatches
  cases where feasible(k) went true -> false somewhere in [1, max]: 0
  feasibility calls: binary 8709 (max 5 per case), brute force 23407
```

> **Go vs Rust:** Identical output (diffed). `Pcg` ports Go's PCG-DXSM and `IntN`, so both programs see the same piles, answer 333,270,263, 29 calls and 8,709 oracle calls. Rust's std has no binary search over an integer range: `partition_point` needs a slice, and building one of 10⁹ candidates is out of the question, so `first_true` is written by hand with `u64::midpoint`, which rounds down and cannot overflow. `p.div_ceil(k)` replaces Go's `(p + k - 1) / k`, and `iter().max()` returns an `Option` where Go's `slices.Max` panics on an empty slice.

**Complexity:** Θ(n log max p) time, Θ(1) extra space. On 10,000 piles of up to 10⁹ the search made 29 feasibility calls (290,000 pile divisions) where a scan from k = 1 would make 333,270,263. The 2,000 random cases had 0 mismatches against brute force and 0 non-monotone predicates.

---

## 9. Ship packing and "minimize the maximum"

`🟡 medium` · *Min capacity / greedy feasibility*

You ship packages in order over at most D days and want the smallest ship capacity that works. That
is a search on the answer again. A greedy left-to-right fill checks one capacity in Θ(n), and a
bigger ship never needs more days, so the check is monotone. The range is `[max w, Σ w]`. Its lower
end is the heaviest package, not 1: search from 1 with a greedy that forgets to check `w > cap`, and
you get a capacity of 1 for ten packages up to 10. The phrase "minimize the maximum" is the tell
for this pattern. The smallest capacity is the smallest possible largest daily load, so the same
function solves split-array-largest-sum. The program checks it against a brute force over every
contiguous split, which also proves the greedy check exact. It costs Θ(n log Σw).

**Steps:**

1. Write the greedy `daysNeeded(w, cap)`, which reports infeasible when a package is heavier than
   `cap`, and search `[max w, Σ w]` with the `firstTrue` loop from example 8.
2. Solve the classic case and print its partition, then search from 1 with the check removed and
   watch it return a capacity that cannot lift the heaviest package.
3. Compare the result with a brute force over every D-way contiguous split on 2,000 PCG cases, and
   check that the feasibility calls never exceed `bits.Len(Σw − max w)`.

```go
package main

import (
	"fmt"
	"math/bits"
	"math/rand/v2"
	"slices"
	"strings"
)

// Ship packages in order, at most D days, one ship per day; the ship's
// capacity is the same every day. What is the smallest capacity that works?
//
// Search the answer again. feasible(cap) = "a greedy left-to-right fill uses
// <= D ships" is monotone (a bigger ship never needs more days), and the
// answer lies in [max(w), sum(w)]:
//   - below max(w) the heaviest package fits on no ship at all,
//   - at sum(w) everything goes in one day.
//
// "Minimize the maximum" is the tell: the smallest cap such that every day's
// load is <= cap is the smallest possible MAXIMUM load. The same function
// solves split-array-largest-sum and painter's partition.

// daysNeeded is the greedy check: fill today's ship until the next package
// would overflow it, then start a new day. ok is false when some package is
// heavier than cap -- no number of days helps.
func daysNeeded(w []int, capacity int) (days int, ok bool) {
	days, load := 1, 0
	for _, x := range w {
		if x > capacity {
			return 0, false
		}
		if load+x > capacity {
			days++
			load = 0
		}
		load += x
	}
	return days, true
}

// daysNaive is the same greedy with the x > capacity check deleted: a
// package heavier than the ship just starts a new day and is "loaded" anyway.
func daysNaive(w []int, capacity int) int {
	days, load := 1, 0
	for _, x := range w {
		if load+x > capacity {
			days++
			load = 0
		}
		load += x
	}
	return days
}

// firstTrue: smallest x in [lo, hi] with f(x) true (f monotone, f(hi) true).
func firstTrue(lo, hi int, f func(int) bool) int {
	for lo < hi {
		mid := lo + (hi-lo)/2
		if f(mid) {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo
}

func sum(w []int) int {
	s := 0
	for _, x := range w {
		s += x
	}
	return s
}

// minCapacity searches [max w, sum w] with the correct greedy.
func minCapacity(w []int, d int) (capacity, calls int) {
	capacity = firstTrue(slices.Max(w), sum(w), func(c int) bool {
		calls++
		days, ok := daysNeeded(w, c)
		return ok && days <= d
	})
	return capacity, calls
}

// partition replays the greedy at capacity c and returns the groups.
func partition(w []int, c int) [][]int {
	var groups [][]int
	start, load := 0, 0
	for i, x := range w {
		if load+x > c {
			groups = append(groups, w[start:i])
			start, load = i, 0
		}
		load += x
	}
	return append(groups, w[start:])
}

func show(groups [][]int) string {
	parts := make([]string, len(groups))
	for i, g := range groups {
		parts[i] = fmt.Sprintf("%v=%d", g, sum(g))
	}
	return strings.Join(parts, " ")
}

// bruteSplit is the oracle: try EVERY way to cut w into exactly d non-empty
// contiguous pieces and return the smallest possible largest piece. (Exactly
// d is the same as at most d when d <= len(w): cutting a piece further never
// raises the maximum.) It counts the splits it tried.
func bruteSplit(w []int, d int, tried *int) int {
	if d == 1 {
		*tried++
		return sum(w)
	}
	best := -1
	for i := 1; i <= len(w)-d+1; i++ { // first piece is w[:i]
		m := max(sum(w[:i]), bruteSplit(w[i:], d-1, tried))
		if best < 0 || m < best {
			best = m
		}
	}
	return best
}

func main() {
	// (a) The classic case.
	w := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	d := 5
	c, calls := minCapacity(w, d)
	fmt.Printf("(a) weights %v, D = %d, capacity searched in [%d, %d]\n", w, d, slices.Max(w), sum(w))
	fmt.Printf("  answer %d after %d feasibility calls\n", c, calls)
	fmt.Printf("  ships: %s\n", show(partition(w, c)))
	days, _ := daysNeeded(w, c-1)
	fmt.Printf("  at capacity %d the greedy needs %d days > %d\n", c-1, days, d)
	// split-array-largest-sum is the SAME function.
	nums := []int{7, 2, 5, 10, 8}
	c2, _ := minCapacity(nums, 2)
	fmt.Printf("  split-array-largest-sum %v into 2: %d, as %s\n", nums, c2, show(partition(nums, c2)))

	// (b) The wrong range: search from 1 with the greedy that has no check.
	fmt.Println()
	d = 10
	naive := firstTrue(1, sum(w), func(c int) bool { return daysNaive(w, c) <= d })
	right, _ := minCapacity(w, d)
	fmt.Printf("(b) same weights, D = %d = n: one package per day fits, so the true answer is max(w) = %d\n", d, slices.Max(w))
	warn := ""
	if naive < slices.Max(w) {
		warn = fmt.Sprintf("  <- cannot lift the %d", slices.Max(w))
	}
	fmt.Printf("  range [1, sum] + greedy without the check: capacity %d%s\n", naive, warn)
	fmt.Printf("  range [max, sum] + correct greedy:         capacity %d\n", right)
	fmt.Printf("  %8s %12s %16s\n", "capacity", "naive days", "correct greedy")
	for _, c := range []int{naive, naive + 1, slices.Max(w) - 1, slices.Max(w)} {
		days, ok := daysNeeded(w, c)
		verdict := fmt.Sprintf("%d days", days)
		if !ok {
			verdict = "infeasible"
		}
		fmt.Printf("  %8d %12d %16s\n", c, daysNaive(w, c), verdict)
	}

	// (c) Oracle + (d) call counts: 2,000 PCG cases against brute force.
	fmt.Println()
	rng := rand.New(rand.NewPCG(1, 2))
	const trials = 2000
	mismatches, tried, totalCalls, overBound := 0, 0, 0, 0
	for range trials {
		n := rng.IntN(10) + 1
		ws := make([]int, n)
		for i := range ws {
			ws[i] = rng.IntN(20) + 1
		}
		dd := rng.IntN(n) + 1
		got, calls := minCapacity(ws, dd)
		if want := bruteSplit(ws, dd, &tried); got != want {
			mismatches++
			if mismatches <= 3 {
				fmt.Println("  MISMATCH", ws, dd, got, want)
			}
		}
		totalCalls += calls
		// the window [max, sum] has sum-max+1 values: at most ceil(log2) of that
		if calls > bits.Len(uint(sum(ws)-slices.Max(ws))) {
			overBound++
		}
	}
	fmt.Printf("(c) %d random cases (1-10 packages of 1-20, D in [1, n]) vs brute force: %d mismatches\n", trials, mismatches)
	fmt.Printf("  brute force tried %d splits; binary search made %d feasibility calls\n", tried, totalCalls)
	fmt.Printf("  cases with more calls than bits.Len(sum - max): %d\n", overBound)

	// (d) One big case: 100,000 packages of 1-500, D = 50.
	big := make([]int, 100_000)
	for i := range big {
		big[i] = rng.IntN(500) + 1
	}
	c, calls = minCapacity(big, 50)
	days, _ = daysNeeded(big, c)
	fmt.Println()
	fmt.Printf("(d) %d packages, sum %d, max %d, D = 50\n", len(big), sum(big), slices.Max(big))
	fmt.Printf("  answer %d (greedy uses %d days; ceil(sum/D) = %d is a lower bound)\n", c, days, (sum(big)+49)/50)
	fmt.Printf("  %d feasibility calls, bits.Len(sum - max) = %d, each call scans %d packages\n",
		calls, bits.Len(uint(sum(big)-slices.Max(big))), len(big))
}
```

**Output:**

```
(a) weights [1 2 3 4 5 6 7 8 9 10], D = 5, capacity searched in [10, 55]
  answer 15 after 5 feasibility calls
  ships: [1 2 3 4 5]=15 [6 7]=13 [8]=8 [9]=9 [10]=10
  at capacity 14 the greedy needs 6 days > 5
  split-array-largest-sum [7 2 5 10 8] into 2: 18, as [7 2 5]=14 [10 8]=18

(b) same weights, D = 10 = n: one package per day fits, so the true answer is max(w) = 10
  range [1, sum] + greedy without the check: capacity 1  <- cannot lift the 10
  range [max, sum] + correct greedy:         capacity 10
  capacity   naive days   correct greedy
         1           10       infeasible
         2           10       infeasible
         9            7       infeasible
        10            7           7 days

(c) 2000 random cases (1-10 packages of 1-20, D in [1, n]) vs brute force: 0 mismatches
  brute force tried 23408 splits; binary search made 9611 feasibility calls
  cases with more calls than bits.Len(sum - max): 0

(d) 100000 packages, sum 25056396, max 500, D = 50
  answer 501314 (greedy uses 50 days; ceil(sum/D) = 501128 is a lower bound)
  25 feasibility calls, bits.Len(sum - max) = 25, each call scans 100000 packages
```

**Rust:**

```rust
// Ship packages in order, at most D days, one ship per day, same capacity
// every day. What is the smallest capacity that works?
//
// Search the answer again. feasible(cap) = "a greedy left-to-right fill uses
// <= D ships" is monotone (a bigger ship never needs more days), and the
// answer lies in [max(w), sum(w)]:
//   - below max(w) the heaviest package fits on no ship at all,
//   - at sum(w) everything goes in one day.
//
// "Minimize the maximum" is the tell: the smallest cap such that every day's
// load is <= cap is the smallest possible MAXIMUM load. The same function
// solves split-array-largest-sum and painter's partition.

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) gives exactly the cases Go generates.
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

/// The greedy check: fill today's ship until the next package would overflow
/// it, then start a new day. `None` when some package is heavier than cap --
/// no number of days helps (Go returns `(0, false)`).
fn days_needed(w: &[u64], cap: u64) -> Option<usize> {
    let (mut days, mut load) = (1, 0);
    for &x in w {
        if x > cap {
            return None;
        }
        if load + x > cap {
            days += 1;
            load = 0;
        }
        load += x;
    }
    Some(days)
}

/// The same greedy with the `x > cap` check deleted: a package heavier than
/// the ship just starts a new day and is "loaded" anyway.
fn days_naive(w: &[u64], cap: u64) -> usize {
    let (mut days, mut load) = (1, 0);
    for &x in w {
        if load + x > cap {
            days += 1;
            load = 0;
        }
        load += x;
    }
    days
}

/// Smallest x in [lo, hi] with f(x) true (f monotone, f(hi) true). Std has
/// no binary search over an integer range, so this is example 1's loop again.
fn first_true(mut lo: u64, mut hi: u64, mut f: impl FnMut(u64) -> bool) -> u64 {
    while lo < hi {
        let mid = lo.midpoint(hi);
        if f(mid) {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    lo
}

/// Searches [max w, sum w] with the correct greedy; `None` for no packages
/// (`max()` on an empty iterator is `None`, where Go's slices.Max panics).
fn min_capacity(w: &[u64], d: usize) -> Option<(u64, usize)> {
    let lo = *w.iter().max()?;
    let hi = w.iter().sum();
    let mut calls = 0;
    let cap = first_true(lo, hi, |c| {
        calls += 1;
        days_needed(w, c).is_some_and(|days| days <= d)
    });
    Some((cap, calls))
}

/// Replays the greedy at capacity c; the groups are borrowed sub-slices of w.
fn partition(w: &[u64], c: u64) -> Vec<&[u64]> {
    let mut groups = Vec::new();
    let (mut start, mut load) = (0, 0);
    for (i, &x) in w.iter().enumerate() {
        if load + x > c {
            groups.push(&w[start..i]);
            (start, load) = (i, 0);
        }
        load += x;
    }
    groups.push(&w[start..]);
    groups
}

fn go_slice(a: &[u64]) -> String {
    let parts: Vec<String> = a.iter().map(u64::to_string).collect();
    format!("[{}]", parts.join(" "))
}

fn show(groups: &[&[u64]]) -> String {
    let parts: Vec<String> = groups
        .iter()
        .map(|g| format!("{}={}", go_slice(g), g.iter().sum::<u64>()))
        .collect();
    parts.join(" ")
}

/// The oracle: try EVERY way to cut w into exactly d non-empty contiguous
/// pieces and return the smallest possible largest piece. (Exactly d equals
/// at most d when d <= len(w): cutting a piece further never raises the max.)
fn brute_split(w: &[u64], d: usize, tried: &mut usize) -> u64 {
    if d == 1 {
        *tried += 1;
        return w.iter().sum();
    }
    (1..=w.len() - d + 1) // the first piece is w[..i]
        .map(|i| {
            let first: u64 = w[..i].iter().sum();
            first.max(brute_split(&w[i..], d - 1, tried))
        })
        .min()
        .expect("at least one way to split")
}

/// Go's bits.Len(x).
fn bits_len(x: u64) -> u32 {
    u64::BITS - x.leading_zeros()
}

fn main() {
    // (a) The classic case.
    let w: Vec<u64> = (1..=10).collect();
    let max_w = *w.iter().max().expect("non-empty");
    let sum_w: u64 = w.iter().sum();
    let d = 5;
    let (c, calls) = min_capacity(&w, d).expect("non-empty");
    println!(
        "(a) weights {}, D = {d}, capacity searched in [{max_w}, {sum_w}]",
        go_slice(&w)
    );
    println!("  answer {c} after {calls} feasibility calls");
    println!("  ships: {}", show(&partition(&w, c)));
    let days = days_needed(&w, c - 1).expect("c - 1 >= max w here");
    println!("  at capacity {} the greedy needs {days} days > {d}", c - 1);
    // split-array-largest-sum is the SAME function.
    let nums = [7, 2, 5, 10, 8];
    let (c2, _) = min_capacity(&nums, 2).expect("non-empty");
    println!(
        "  split-array-largest-sum {} into 2: {c2}, as {}",
        go_slice(&nums),
        show(&partition(&nums, c2))
    );

    // (b) The wrong range: search from 1 with the greedy that has no check.
    println!();
    let d = 10;
    let naive = first_true(1, sum_w, |c| days_naive(&w, c) <= d);
    let (right, _) = min_capacity(&w, d).expect("non-empty");
    println!(
        "(b) same weights, D = {d} = n: one package per day fits, so the true answer is max(w) = {max_w}"
    );
    let warn = if naive < max_w {
        format!("  <- cannot lift the {max_w}")
    } else {
        String::new()
    };
    println!("  range [1, sum] + greedy without the check: capacity {naive}{warn}");
    println!("  range [max, sum] + correct greedy:         capacity {right}");
    println!(
        "  {:>8} {:>12} {:>16}",
        "capacity", "naive days", "correct greedy"
    );
    for c in [naive, naive + 1, max_w - 1, max_w] {
        let verdict = days_needed(&w, c).map_or("infeasible".to_string(), |d| format!("{d} days"));
        println!("  {c:>8} {:>12} {verdict:>16}", days_naive(&w, c));
    }

    // (c) Oracle + (d) call counts: 2,000 PCG cases against brute force.
    println!();
    let mut rng = Pcg::new(1, 2);
    const TRIALS: usize = 2000;
    let (mut mismatches, mut tried, mut total_calls, mut over_bound) = (0, 0, 0, 0);
    for _ in 0..TRIALS {
        let n = rng.int_n(10) + 1;
        let ws: Vec<u64> = (0..n).map(|_| rng.int_n(20) + 1).collect();
        let dd = rng.int_n(n) as usize + 1;
        let (got, calls) = min_capacity(&ws, dd).expect("n >= 1");
        let want = brute_split(&ws, dd, &mut tried);
        if got != want {
            mismatches += 1;
            if mismatches <= 3 {
                println!("  MISMATCH {} {dd} {got} {want}", go_slice(&ws));
            }
        }
        total_calls += calls;
        // the window [max, sum] has sum-max+1 values: at most ceil(log2) of that
        let span = ws.iter().sum::<u64>() - ws.iter().max().expect("n >= 1");
        if calls > bits_len(span) as usize {
            over_bound += 1;
        }
    }
    println!(
        "(c) {TRIALS} random cases (1-10 packages of 1-20, D in [1, n]) vs brute force: {mismatches} mismatches"
    );
    println!(
        "  brute force tried {tried} splits; binary search made {total_calls} feasibility calls"
    );
    println!("  cases with more calls than bits.Len(sum - max): {over_bound}");

    // (d) One big case: 100,000 packages of 1-500, D = 50.
    let big: Vec<u64> = (0..100_000).map(|_| rng.int_n(500) + 1).collect();
    let (c, calls) = min_capacity(&big, 50).expect("non-empty");
    let days = days_needed(&big, c).expect("c >= max");
    let (sum_b, max_b) = (
        big.iter().sum::<u64>(),
        *big.iter().max().expect("non-empty"),
    );
    println!();
    println!(
        "(d) {} packages, sum {sum_b}, max {max_b}, D = 50",
        big.len()
    );
    println!(
        "  answer {c} (greedy uses {days} days; ceil(sum/D) = {} is a lower bound)",
        sum_b.div_ceil(50)
    );
    println!(
        "  {calls} feasibility calls, bits.Len(sum - max) = {}, each call scans {} packages",
        bits_len(sum_b - max_b),
        big.len()
    );
}
```

**Rust output:**

```
(a) weights [1 2 3 4 5 6 7 8 9 10], D = 5, capacity searched in [10, 55]
  answer 15 after 5 feasibility calls
  ships: [1 2 3 4 5]=15 [6 7]=13 [8]=8 [9]=9 [10]=10
  at capacity 14 the greedy needs 6 days > 5
  split-array-largest-sum [7 2 5 10 8] into 2: 18, as [7 2 5]=14 [10 8]=18

(b) same weights, D = 10 = n: one package per day fits, so the true answer is max(w) = 10
  range [1, sum] + greedy without the check: capacity 1  <- cannot lift the 10
  range [max, sum] + correct greedy:         capacity 10
  capacity   naive days   correct greedy
         1           10       infeasible
         2           10       infeasible
         9            7       infeasible
        10            7           7 days

(c) 2000 random cases (1-10 packages of 1-20, D in [1, n]) vs brute force: 0 mismatches
  brute force tried 23408 splits; binary search made 9611 feasibility calls
  cases with more calls than bits.Len(sum - max): 0

(d) 100000 packages, sum 25056396, max 500, D = 50
  answer 501314 (greedy uses 50 days; ceil(sum/D) = 501128 is a lower bound)
  25 feasibility calls, bits.Len(sum - max) = 25, each call scans 100000 packages
```

> **Go vs Rust:** Identical output (diffed): the same PCG cases, 23,408 brute-force splits, 9,611 calls, and answer 501314 in 25 calls. The Go `(days, ok)` pair becomes `Option<usize>`, and an empty input makes `min_capacity` return `None` because `iter().max()` is an `Option`, while Go's `slices.Max` panics. `partition` returns borrowed sub-slices (`Vec<&[u64]>`), just as Go's groups share `w`'s backing array, but the borrow checker stops `w` changing while they are alive. The range search is hand-written again, because std has no binary search over an integer range.

**Complexity:** Θ(n log(Σw − max w)) time, Θ(1) extra space; the greedy check is Θ(n) and exact. The 2,000 PCG cases had 0 mismatches against a brute force that tried 23,408 splits, and no case needed more than `bits.Len(Σw − max w)` calls. The 100,000-package case took 25 calls. Searching from 1 with the check removed returns capacity 1 for packages up to 10.

---

## 10. Two kinds of sorted matrix

`🟡 medium` · *2-D search*

"Sorted matrix" means two different things. In a **row-major sorted** matrix every row starts after the
previous one ends, so reading it row by row gives one sorted array of m·n elements: map virtual index
`i → (i / cols, i % cols)` and run one binary search, Θ(log mn). In a **row- and column-sorted** matrix the
rows overlap, the row-by-row sequence goes up and down, and that flat search silently misses elements. There
the tool is the **staircase**: start at the top-right corner, where one comparison throws away a whole row
(too small: go down) or a whole column (too big: go left), Θ(m + n). Or search every row, Θ(m log n). Which
of those two wins depends on the shape, and the program counts element reads to show it.

**Steps:**

1. Build a 4×5 row-major matrix and search it as a flat array; check every target against a full scan.
2. Run the same flat search on LeetCode 240's row/column-sorted matrix and list the targets it gets wrong; then trace the staircase.
3. Count element reads on 10⁶-element matrices of two shapes, 1000×1000 and 10×100,000, for the staircase, per-row search and the flat search on the row-major twin.

```go
package main

import (
	"fmt"
	"math/bits"
	"slices"
)

// Two kinds of "sorted matrix" look alike and need different algorithms.
//
//   row-major sorted:  every row is sorted AND each row starts after the
//                      previous one ends. Read it row by row and you get one
//                      sorted array of m*n elements -> one binary search.
//   row/col sorted:    every row is sorted and every column is sorted, but a
//                      row may start below the previous row's end. Read row by
//                      row and the sequence goes up and down -> the flat search
//                      is wrong; use the staircase (or one search per row).
//
// Every function returns how many matrix elements it READ (probes), so the
// costs below are counted, not guessed.

// lowerBound: the half-open template over n virtual elements, read through
// get(i). Returns the first i with get(i) >= t (or n) and how many reads it took.
func lowerBound(n int, get func(int) int, t int) (int, int) {
	lo, hi, probes := 0, n, 0
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		probes++
		if get(mid) >= t {
			hi = mid
		} else {
			lo = mid + 1
		}
	}
	return lo, probes
}

// flatSearch treats an m x cols matrix as ONE sorted array of m*cols elements:
// virtual index i lives at (i/cols, i%cols). One lower bound, then one final
// read to decide "found". Correct ONLY for a row-major sorted matrix.
func flatSearch(a [][]int, t int) (row, col int, found bool, probes int) {
	cols := len(a[0])
	at := func(i int) int { return a[i/cols][i%cols] }
	i, probes := lowerBound(len(a)*cols, at, t)
	if i < len(a)*cols {
		probes++
		if at(i) == t {
			return i / cols, i % cols, true, probes
		}
	}
	return -1, -1, false, probes
}

// staircase starts at the top-right corner. Everything left of it in its row
// is smaller, everything below it in its column is larger, so one comparison
// throws away a whole row (too small: go down) or a whole column (too big: go
// left). At most m+n-1 reads.
func staircase(a [][]int, t int) (row, col int, found bool, probes int) {
	r, c := 0, len(a[0])-1
	for r < len(a) && c >= 0 {
		probes++
		switch v := a[r][c]; {
		case v == t:
			return r, c, true, probes
		case v > t:
			c-- // the whole column below is >= v > t
		default:
			r++ // the whole row to the left is <= v < t
		}
	}
	return -1, -1, false, probes
}

// perRow runs a lower bound on every row until one holds t: Theta(m log n).
func perRow(a [][]int, t int) (row, col int, found bool, probes int) {
	for r, rowVals := range a {
		i, p := lowerBound(len(rowVals), func(j int) int { return rowVals[j] }, t)
		probes += p
		if i < len(rowVals) {
			probes++
			if rowVals[i] == t {
				return r, i, true, probes
			}
		}
	}
	return -1, -1, false, probes
}

// isRowMajor is the test that tells the two kinds apart: every row sorted and
// each row's last element <= the next row's first.
func isRowMajor(a [][]int) bool {
	for r := range a {
		if !slices.IsSorted(a[r]) || (r+1 < len(a) && a[r][len(a[r])-1] > a[r+1][0]) {
			return false
		}
	}
	return true
}

func contains(a [][]int, t int) bool { // the oracle: read everything
	for _, row := range a {
		if slices.Contains(row, t) {
			return true
		}
	}
	return false
}

// bitsLen(n) = bits.Len(n) = ceil(log2(n+1)): the most probes a lower bound
// over n elements can make (lesson example 1).
func bitsLen(n int) int { return bits.Len(uint(n)) }

func main() {
	// (a) row-major: 4 x 5, values 2, 5, 8, ..., 59.
	rm := make([][]int, 4)
	for r := range rm {
		rm[r] = make([]int, 5)
		for c := range rm[r] {
			rm[r][c] = 3*(r*5+c) + 2
		}
	}
	fmt.Println("(a) row-major sorted 4 x 5 -- one search over 20 virtual indices")
	for _, row := range rm {
		fmt.Printf("    %3d\n", row)
	}
	fmt.Printf("    row-major test (row ends <= next row starts): %v\n", isRowMajor(rm))
	for _, t := range []int{2, 29, 59, 30, 1, 60} {
		r, c, ok, p := flatSearch(rm, t)
		if ok {
			fmt.Printf("    find %2d -> (%d,%d)  %d probes\n", t, r, c, p)
		} else {
			fmt.Printf("    find %2d -> absent %d probes\n", t, p)
		}
	}
	wrong, maxP := 0, 0
	for t := 0; t <= 61; t++ {
		_, _, ok, p := flatSearch(rm, t)
		if ok != contains(rm, t) {
			wrong++
		}
		maxP = max(maxP, p)
	}
	fmt.Printf("    every target 0..61: %d wrong, max probes %d (lower bound %d + 1 final read)\n",
		wrong, maxP, bitsLen(20))

	// (b) row/col sorted: the matrix from LeetCode 240.
	rc := [][]int{
		{1, 4, 7, 11, 15},
		{2, 5, 8, 12, 19},
		{3, 6, 9, 16, 22},
		{10, 13, 14, 17, 24},
		{18, 21, 23, 26, 30},
	}
	fmt.Println()
	fmt.Println("(b) row- and column-sorted 5 x 5 -- rows overlap, so it is NOT one sorted array")
	for _, row := range rc {
		fmt.Printf("    %3d\n", row)
	}
	fmt.Printf("    row-major test: %v (row 0 ends at %d, row 1 starts at %d)\n",
		isRowMajor(rc), rc[0][4], rc[1][0])
	var flatWrong []int
	stairWrong, stairMax := 0, 0
	for t := 0; t <= 31; t++ {
		want := contains(rc, t)
		if _, _, ok, _ := flatSearch(rc, t); ok != want {
			flatWrong = append(flatWrong, t)
		}
		_, _, ok, p := staircase(rc, t)
		if ok != want {
			stairWrong++
		}
		stairMax = max(stairMax, p)
	}
	fmt.Printf("    flat search, every target 0..31: wrong on %d: %v\n", len(flatWrong), flatWrong)
	_, _, ok, _ := flatSearch(rc, flatWrong[0])
	fmt.Printf("      e.g. %d is in the matrix (contains=%v) but flat search says found=%v\n",
		flatWrong[0], contains(rc, flatWrong[0]), ok)
	fmt.Printf("    staircase, every target 0..31: wrong on %d, max probes %d (m+n-1 = %d)\n",
		stairWrong, stairMax, 5+5-1)

	fmt.Println("    staircase trace for 14 (start top-right):")
	r, c := 0, 4
	for {
		v := rc[r][c]
		if v == 14 {
			fmt.Printf("      (%d,%d)=%2d  == 14  found\n", r, c, v)
			break
		}
		if v > 14 {
			fmt.Printf("      (%d,%d)=%2d  > 14   drop column %d, go left\n", r, c, v, c)
			c--
		} else {
			fmt.Printf("      (%d,%d)=%2d  < 14   drop row %d, go down\n", r, c, v, r)
			r++
		}
	}

	// (c) Big matrices, counted. a[i][j] = 2*(i + 2j): rows and columns
	// increase, values are even (odd targets are absent), with duplicates.
	// The row-major twin holds the same values, sorted and reshaped.
	fmt.Println()
	fmt.Println("(c) probes per query on 10^6-element matrices (same values in both kinds)")
	fmt.Printf("    %-11s %-24s %8s %9s %8s  %s\n", "shape", "method", "queries", "avg", "max", "bound")
	totalWrong := 0
	for _, shape := range [][2]int{{1000, 1000}, {10, 100_000}} {
		m, n := shape[0], shape[1]
		a := make([][]int, m)
		all := make([]int, 0, m*n)
		for i := range a {
			a[i] = make([]int, n)
			for j := range a[i] {
				a[i][j] = 2 * (i + 2*j)
			}
			all = append(all, a[i]...)
		}
		slices.Sort(all)
		flat := make([][]int, m) // row-major twin: reshape the sorted values
		for i := range flat {
			flat[i] = all[i*n : (i+1)*n]
		}
		maxV := all[len(all)-1]
		present := make([]bool, maxV+1) // brute-force oracle: one full scan
		for _, v := range all {
			present[v] = true
		}
		step := (maxV+2)/2000 | 1 // odd, so targets alternate even/odd
		var targets []int
		for t := -1; t <= maxV+1; t += step {
			targets = append(targets, t)
		}

		type method struct {
			name  string
			mat   [][]int
			f     func([][]int, int) (int, int, bool, int)
			bound string
		}
		methods := []method{
			{"staircase (row/col)", a, staircase, fmt.Sprintf("m+n-1=%d", m+n-1)},
			{"per-row search (row/col)", a, perRow, fmt.Sprintf("m*(%d+1)=%d", bitsLen(n), m*(bitsLen(n)+1))},
			{"flat search (row-major)", flat, flatSearch, fmt.Sprintf("%d+1=%d", bitsLen(m*n), bitsLen(m*n)+1)},
		}
		label := fmt.Sprintf("%dx%d", m, n)
		avgs := make([]float64, len(methods))
		for k, me := range methods {
			total, mx, wrong := 0, 0, 0
			for _, t := range targets {
				r, c, ok, p := me.f(me.mat, t)
				want := t >= 0 && t <= maxV && present[t]
				if ok != want || (ok && me.mat[r][c] != t) {
					wrong++
				}
				total += p
				mx = max(mx, p)
			}
			avgs[k] = float64(total) / float64(len(targets))
			fmt.Printf("    %-11s %-24s %8d %9.1f %8d  %s\n", label, me.name, len(targets),
				avgs[k], mx, me.bound)
			totalWrong += wrong
			label = ""
		}
		// Which row/col method wins depends on the shape: m+n vs m*log n.
		win, lose := "the staircase", "per-row search"
		ratio := avgs[1] / avgs[0]
		if avgs[1] < avgs[0] {
			win, lose, ratio = lose, win, avgs[0]/avgs[1]
		}
		fmt.Printf("    %-11s -> row/col matrix: %s reads %.1fx fewer elements than %s\n", "", win, ratio, lose)
	}
	fmt.Printf("    every answer above checked against a full scan: %d wrong\n", totalWrong)
}
```

**Output:**

```
(a) row-major sorted 4 x 5 -- one search over 20 virtual indices
    [  2   5   8  11  14]
    [ 17  20  23  26  29]
    [ 32  35  38  41  44]
    [ 47  50  53  56  59]
    row-major test (row ends <= next row starts): true
    find  2 -> (0,0)  6 probes
    find 29 -> (1,4)  5 probes
    find 59 -> (3,4)  5 probes
    find 30 -> absent 5 probes
    find  1 -> absent 6 probes
    find 60 -> absent 4 probes
    every target 0..61: 0 wrong, max probes 6 (lower bound 5 + 1 final read)

(b) row- and column-sorted 5 x 5 -- rows overlap, so it is NOT one sorted array
    [  1   4   7  11  15]
    [  2   5   8  12  19]
    [  3   6   9  16  22]
    [ 10  13  14  17  24]
    [ 18  21  23  26  30]
    row-major test: false (row 0 ends at 15, row 1 starts at 2)
    flat search, every target 0..31: wrong on 17: [2 3 5 6 7 9 10 11 12 13 15 16 18 19 21 22 23]
      e.g. 2 is in the matrix (contains=true) but flat search says found=false
    staircase, every target 0..31: wrong on 0, max probes 9 (m+n-1 = 9)
    staircase trace for 14 (start top-right):
      (0,4)=15  > 14   drop column 4, go left
      (0,3)=11  < 14   drop row 0, go down
      (1,3)=12  < 14   drop row 1, go down
      (2,3)=16  > 14   drop column 3, go left
      (2,2)= 9  < 14   drop row 2, go down
      (3,2)=14  == 14  found

(c) probes per query on 10^6-element matrices (same values in both kinds)
    shape       method                    queries       avg      max  bound
    1000x1000   staircase (row/col)          1999    1000.5     1999  m+n-1=1999
                per-row search (row/col)     1999    6083.9    11000  m*(10+1)=11000
                flat search (row-major)      1999      20.9       21  20+1=21
                -> row/col matrix: the staircase reads 6.1x fewer elements than per-row search
    10x100000   staircase (row/col)          1991   50008.9   100000  m+n-1=100009
                per-row search (row/col)     1991     101.7      180  m*(17+1)=180
                flat search (row-major)      1991      21.0       21  20+1=21
                -> row/col matrix: per-row search reads 491.5x fewer elements than the staircase
    every answer above checked against a full scan: 0 wrong
```

**Rust:**

```rust
// Two kinds of "sorted matrix" look alike and need different algorithms.
//
//   row-major sorted:  every row is sorted AND each row starts after the
//                      previous one ends. Read it row by row and you get one
//                      sorted array of m*n elements -> one binary search.
//   row/col sorted:    every row is sorted and every column is sorted, but a
//                      row may start below the previous row's end. Read row by
//                      row and the sequence goes up and down -> the flat search
//                      is wrong; use the staircase (or one search per row).
//
// Representation follows the kind: a row-major matrix is stored FLAT, one
// `Vec<i64>` plus a column count (rows are `chunks_exact(cols)`), because it
// really is one sorted array. The row/col matrix is a `Vec<Vec<i64>>`.
// Every search returns how many matrix elements it READ (probes), so the costs
// below are counted, not guessed. A hit is `Some((row, col))`, not -1/-1.

type Hit = Option<(usize, usize)>;

/// The half-open template over n virtual elements, read through `get(i)`.
/// Returns the first i with get(i) >= t (or n) and how many reads it took.
fn lower_bound(n: usize, get: impl Fn(usize) -> i64, t: i64) -> (usize, usize) {
    let (mut lo, mut hi, mut probes) = (0, n, 0);
    while lo < hi {
        let mid = lo.midpoint(hi); // usize::midpoint: no overflow, rounds down
        probes += 1;
        if get(mid) >= t {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    (lo, probes)
}

/// One lower bound over the flat storage, then one final read to decide
/// "found". Virtual index i is (i / cols, i % cols). Correct ONLY when the
/// flat sequence is sorted, i.e. for a row-major sorted matrix.
fn flat_search(cells: &[i64], cols: usize, t: i64) -> (Hit, usize) {
    let (i, mut probes) = lower_bound(cells.len(), |i| cells[i], t);
    if i < cells.len() {
        probes += 1;
        if cells[i] == t {
            return (Some((i / cols, i % cols)), probes);
        }
    }
    (None, probes)
}

/// Start at the top-right corner. Everything left of it in its row is smaller,
/// everything below it in its column is larger, so one comparison throws away
/// a whole row (too small: go down) or a whole column (too big: go left).
/// At most m+n-1 reads.
fn staircase(a: &[Vec<i64>], t: i64) -> (Hit, usize) {
    let (mut r, mut c, mut probes) = (0, a[0].len() - 1, 0);
    while r < a.len() {
        probes += 1;
        let v = a[r][c];
        if v == t {
            return (Some((r, c)), probes);
        } else if v > t {
            // Go writes `c--` and tests `c >= 0`. A usize cannot go below 0:
            // `c - 1` at c = 0 would panic in a debug build (wrap with -O), so
            // ask checked_sub and stop when there is no column left.
            match c.checked_sub(1) {
                Some(left) => c = left,
                None => break,
            }
        } else {
            r += 1; // the whole row to the left is <= v < t
        }
    }
    (None, probes)
}

/// A lower bound on every row until one holds t: Theta(m log n).
fn per_row(a: &[Vec<i64>], t: i64) -> (Hit, usize) {
    let mut probes = 0;
    for (r, row) in a.iter().enumerate() {
        let (i, p) = lower_bound(row.len(), |j| row[j], t);
        probes += p;
        if i < row.len() {
            probes += 1;
            if row[i] == t {
                return (Some((r, i)), probes);
            }
        }
    }
    (None, probes)
}

/// The test that tells the two kinds apart: every row sorted and each row's
/// last element <= the next row's first.
fn is_row_major<R: AsRef<[i64]>>(rows: &[R]) -> bool {
    rows.iter().all(|r| r.as_ref().is_sorted())
        && rows
            .windows(2)
            .all(|w| w[0].as_ref().last() <= w[1].as_ref().first())
}

/// ceil(log2(n+1)), Go's bits.Len: the most probes a lower bound over n
/// elements can make (lesson example 1).
fn bits_len(n: usize) -> u32 {
    usize::BITS - n.leading_zeros()
}

/// Go's %3d on a slice: `[  2   5]`.
fn go_row(row: &[i64]) -> String {
    let cells: Vec<String> = row.iter().map(|v| format!("{v:3}")).collect();
    format!("[{}]", cells.join(" "))
}

fn go_list(xs: &[i64]) -> String {
    let cells: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", cells.join(" "))
}

fn main() {
    // (a) row-major: 4 x 5, values 2, 5, 8, ..., 59, stored flat.
    let cols = 5;
    let rm: Vec<i64> = (0..20).map(|i| 3 * i + 2).collect();
    println!("(a) row-major sorted 4 x 5 -- one search over 20 virtual indices");
    for row in rm.chunks_exact(cols) {
        println!("    {}", go_row(row));
    }
    let rows: Vec<&[i64]> = rm.chunks_exact(cols).collect();
    println!(
        "    row-major test (row ends <= next row starts): {}",
        is_row_major(&rows)
    );
    for t in [2, 29, 59, 30, 1, 60] {
        match flat_search(&rm, cols, t) {
            (Some((r, c)), p) => println!("    find {t:2} -> ({r},{c})  {p} probes"),
            (None, p) => println!("    find {t:2} -> absent {p} probes"),
        }
    }
    let (mut wrong, mut max_p) = (0, 0);
    for t in 0..=61 {
        let (hit, p) = flat_search(&rm, cols, t);
        if hit.is_some() != rm.contains(&t) {
            wrong += 1;
        }
        max_p = max_p.max(p);
    }
    println!(
        "    every target 0..61: {wrong} wrong, max probes {max_p} (lower bound {} + 1 final read)",
        bits_len(20)
    );

    // (b) row/col sorted: the matrix from LeetCode 240.
    let rc: Vec<Vec<i64>> = vec![
        vec![1, 4, 7, 11, 15],
        vec![2, 5, 8, 12, 19],
        vec![3, 6, 9, 16, 22],
        vec![10, 13, 14, 17, 24],
        vec![18, 21, 23, 26, 30],
    ];
    let contains = |t: i64| rc.iter().any(|row| row.contains(&t)); // the oracle
    println!();
    println!("(b) row- and column-sorted 5 x 5 -- rows overlap, so it is NOT one sorted array");
    for row in &rc {
        println!("    {}", go_row(row));
    }
    println!(
        "    row-major test: {} (row 0 ends at {}, row 1 starts at {})",
        is_row_major(&rc),
        rc[0][4],
        rc[1][0]
    );
    let rc_flat = rc.concat(); // the same cells, read row by row
    let mut flat_wrong = Vec::new();
    let (mut stair_wrong, mut stair_max) = (0, 0);
    for t in 0..=31 {
        let want = contains(t);
        if flat_search(&rc_flat, 5, t).0.is_some() != want {
            flat_wrong.push(t);
        }
        let (hit, p) = staircase(&rc, t);
        if hit.is_some() != want {
            stair_wrong += 1;
        }
        stair_max = stair_max.max(p);
    }
    println!(
        "    flat search, every target 0..31: wrong on {}: {}",
        flat_wrong.len(),
        go_list(&flat_wrong)
    );
    let first = flat_wrong[0];
    println!(
        "      e.g. {first} is in the matrix (contains={}) but flat search says found={}",
        contains(first),
        flat_search(&rc_flat, 5, first).0.is_some()
    );
    println!(
        "    staircase, every target 0..31: wrong on {stair_wrong}, max probes {stair_max} (m+n-1 = {})",
        5 + 5 - 1
    );

    println!("    staircase trace for 14 (start top-right):");
    let (mut r, mut c) = (0, 4);
    loop {
        let v = rc[r][c];
        if v == 14 {
            println!("      ({r},{c})={v:2}  == 14  found");
            break;
        }
        if v > 14 {
            println!("      ({r},{c})={v:2}  > 14   drop column {c}, go left");
            c -= 1; // never at c = 0 here: 14 is found first
        } else {
            println!("      ({r},{c})={v:2}  < 14   drop row {r}, go down");
            r += 1;
        }
    }

    // (c) Big matrices, counted. a[i][j] = 2*(i + 2j): rows and columns
    // increase, values are even (odd targets are absent), with duplicates.
    // The row-major twin holds the same values: concatenate, sort -- and that
    // flat Vec already IS the row-major matrix; no reshaping needed.
    println!();
    println!("(c) probes per query on 10^6-element matrices (same values in both kinds)");
    println!(
        "    {:<11} {:<24} {:>8} {:>9} {:>8}  bound",
        "shape", "method", "queries", "avg", "max"
    );
    let mut total_wrong = 0;
    for (m, n) in [(1000usize, 1000usize), (10, 100_000)] {
        let a: Vec<Vec<i64>> = (0..m)
            .map(|i| (0..n).map(|j| 2 * (i + 2 * j) as i64).collect())
            .collect();
        let mut all = a.concat();
        all.sort_unstable();
        let max_v = *all.last().expect("matrix is not empty");
        let mut present = vec![false; max_v as usize + 1]; // brute-force oracle
        for &v in &all {
            present[v as usize] = true;
        }
        let step = ((max_v + 2) / 2000) | 1; // odd, so targets alternate even/odd
        let targets: Vec<i64> = (-1..=max_v + 1).step_by(step as usize).collect();

        // (name, bound, search, read-back of a reported cell for the check)
        type Search<'a> = &'a dyn Fn(i64) -> (Hit, usize);
        type Cell<'a> = &'a dyn Fn(usize, usize) -> i64;
        let stair = |t| staircase(&a, t);
        let rows = |t| per_row(&a, t);
        let flat = |t| flat_search(&all, n, t);
        let rc_cell = |r: usize, c: usize| a[r][c];
        let rm_cell = |r: usize, c: usize| all[r * n + c];
        let methods: [(&str, String, Search, Cell); 3] = [
            (
                "staircase (row/col)",
                format!("m+n-1={}", m + n - 1),
                &stair,
                &rc_cell,
            ),
            (
                "per-row search (row/col)",
                format!("m*({}+1)={}", bits_len(n), m * (bits_len(n) as usize + 1)),
                &rows,
                &rc_cell,
            ),
            (
                "flat search (row-major)",
                format!("{}+1={}", bits_len(m * n), bits_len(m * n) + 1),
                &flat,
                &rm_cell,
            ),
        ];
        let mut label = format!("{m}x{n}");
        let mut avgs = [0.0; 3];
        for (k, (name, bound, search, cell)) in methods.iter().enumerate() {
            let (mut total, mut mx, mut wrong) = (0, 0, 0);
            for &t in &targets {
                let (hit, p) = search(t);
                let want = (0..=max_v).contains(&t) && present[t as usize];
                let right = match hit {
                    Some((r, c)) => want && cell(r, c) == t,
                    None => !want,
                };
                if !right {
                    wrong += 1;
                }
                total += p;
                mx = mx.max(p);
            }
            avgs[k] = total as f64 / targets.len() as f64;
            println!(
                "    {label:<11} {name:<24} {:>8} {:>9.1} {mx:>8}  {bound}",
                targets.len(),
                avgs[k]
            );
            total_wrong += wrong;
            label.clear();
        }
        // Which row/col method wins depends on the shape: m+n vs m*log n.
        let (win, lose, ratio) = if avgs[1] < avgs[0] {
            ("per-row search", "the staircase", avgs[0] / avgs[1])
        } else {
            ("the staircase", "per-row search", avgs[1] / avgs[0])
        };
        println!(
            "    {:<11} -> row/col matrix: {win} reads {ratio:.1}x fewer elements than {lose}",
            ""
        );
    }
    println!("    every answer above checked against a full scan: {total_wrong} wrong");
}
```

**Rust output:**

```
(a) row-major sorted 4 x 5 -- one search over 20 virtual indices
    [  2   5   8  11  14]
    [ 17  20  23  26  29]
    [ 32  35  38  41  44]
    [ 47  50  53  56  59]
    row-major test (row ends <= next row starts): true
    find  2 -> (0,0)  6 probes
    find 29 -> (1,4)  5 probes
    find 59 -> (3,4)  5 probes
    find 30 -> absent 5 probes
    find  1 -> absent 6 probes
    find 60 -> absent 4 probes
    every target 0..61: 0 wrong, max probes 6 (lower bound 5 + 1 final read)

(b) row- and column-sorted 5 x 5 -- rows overlap, so it is NOT one sorted array
    [  1   4   7  11  15]
    [  2   5   8  12  19]
    [  3   6   9  16  22]
    [ 10  13  14  17  24]
    [ 18  21  23  26  30]
    row-major test: false (row 0 ends at 15, row 1 starts at 2)
    flat search, every target 0..31: wrong on 17: [2 3 5 6 7 9 10 11 12 13 15 16 18 19 21 22 23]
      e.g. 2 is in the matrix (contains=true) but flat search says found=false
    staircase, every target 0..31: wrong on 0, max probes 9 (m+n-1 = 9)
    staircase trace for 14 (start top-right):
      (0,4)=15  > 14   drop column 4, go left
      (0,3)=11  < 14   drop row 0, go down
      (1,3)=12  < 14   drop row 1, go down
      (2,3)=16  > 14   drop column 3, go left
      (2,2)= 9  < 14   drop row 2, go down
      (3,2)=14  == 14  found

(c) probes per query on 10^6-element matrices (same values in both kinds)
    shape       method                    queries       avg      max  bound
    1000x1000   staircase (row/col)          1999    1000.5     1999  m+n-1=1999
                per-row search (row/col)     1999    6083.9    11000  m*(10+1)=11000
                flat search (row-major)      1999      20.9       21  20+1=21
                -> row/col matrix: the staircase reads 6.1x fewer elements than per-row search
    10x100000   staircase (row/col)          1991   50008.9   100000  m+n-1=100009
                per-row search (row/col)     1991     101.7      180  m*(17+1)=180
                flat search (row-major)      1991      21.0       21  20+1=21
                -> row/col matrix: per-row search reads 491.5x fewer elements than the staircase
    every answer above checked against a full scan: 0 wrong
```

> **Go vs Rust:** Identical output, diffed line by line: every probe count belongs to the algorithm, and the hand-written `lower_bound` uses the same loop as Go's. The storage follows the kind: the row-major matrix is one flat `Vec<i64>` read through `chunks_exact(cols)`, so in (c) the row-major twin is just the concatenated, sorted cells with no reshaping, while the row/col matrix stays a `Vec<Vec<i64>>`. Go's staircase steps `c--` to -1 and stops on `c >= 0`. A `usize` cannot hold -1 (`0 - 1` panics without `-O` and wraps with it), so the Rust loop asks `c.checked_sub(1)` and stops on `None`. Hits are `Option<(usize, usize)>`, not `-1, -1`.

**Complexity:** row-major: Θ(log mn) — at most 21 reads on 10⁶ elements (20 loop probes + 1 final read) · row/col-sorted: staircase Θ(m + n) (max 1,999 reads at 1000×1000, 100,000 at 10×100,000), per-row Θ(m log n) (max 11,000 and 180) — the staircase reads 6.1× fewer on the square, per-row 491.5× fewer on the wide one · the flat trick is valid only for the first kind: on LeetCode 240's matrix it misses 17 of 32 targets.

---

## 11. Finding a peak in unsorted data

`🟡 medium` · *Binary search without sorting*

Binary search needs a rule that says which half still holds an answer, not sorted data. A peak is an element
larger than both neighbours (outside the array counts as −∞). If `a[mid] < a[mid+1]`, walking uphill to the
right must top out somewhere, so a peak lies right of `mid`; otherwise one lies at or left of it. The
invariant is about the window's ends (the left end climbs, the right end falls), so the search works even
though neither "is a peak" nor "`a[i] > a[i+1]`" is monotone on unsorted data. That is Θ(log n)
comparisons, but it finds *a* peak, not the maximum, and on random arrays a plain scan is often cheaper on
average. The binary search's advantage is its worst case. On a **mountain** (bitonic) array the only peak
*is* the maximum, so peak + one ascending search + one descending search finds any value in Θ(log n).

**Steps:**

1. Trace `findPeak` on a 12-element array with four peaks, checking the invariant after every update.
2. Run it on 10,000 random permutations (PCG) and check each answer against the definition. Count how often it is the global maximum, and compare its comparisons with a left-to-right scan's.
3. Build a 10⁶-element mountain, find its top, then search both halves with `sort.Search` (`>=` ascending, `<=` descending, as its doc says) and check against a full scan and the stdlib.

```go
package main

import (
	"cmp"
	"fmt"
	"math/bits"
	"math/rand/v2"
	"slices"
	"sort"
)

// Binary search does not need sorted data. It needs a rule that says which
// half still holds an answer. For a PEAK (an element larger than its
// neighbours; outside the array counts as -infinity):
//
//   a[mid] < a[mid+1]  -> walking uphill to the right must top out somewhere
//                         right of mid: a peak lies in [mid+1, hi]
//   otherwise          -> a[mid] > a[mid+1] (distinct values): a peak lies
//                         in [lo, mid]
//
// Invariant on the window [lo, hi]:
//   left end climbs:   lo == 0   || a[lo-1] < a[lo]
//   right end falls:   hi == n-1 || a[hi]   > a[hi+1]
// A window with a rising left end and a falling right end contains a peak, and
// when it shrinks to one element, that element IS a peak.

// findPeak returns a peak index and the number of a[mid] vs a[mid+1]
// comparisons. Window [lo, hi] closed, so mid+1 <= hi is always in range.
func findPeak(a []int) (int, int) {
	lo, hi, comps := 0, len(a)-1, 0
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		comps++
		if a[mid] < a[mid+1] {
			lo = mid + 1
		} else {
			hi = mid
		}
	}
	return lo, comps
}

func isPeak(a []int, i int) bool {
	return (i == 0 || a[i-1] < a[i]) && (i == len(a)-1 || a[i] > a[i+1])
}

func invariant(a []int, lo, hi int) string {
	if (lo == 0 || a[lo-1] < a[lo]) && (hi == len(a)-1 || a[hi] > a[hi+1]) {
		return "invariant holds"
	}
	return "INVARIANT BROKEN"
}

// bitString prints a predicate over 0..n-1 as 0/1.
func bitString(n int, p func(int) bool) string {
	b := make([]byte, n)
	for i := range b {
		b[i] = '0'
		if p(i) {
			b[i] = '1'
		}
	}
	return string(b)
}

func main() {
	// (a) A trace on unsorted data with four peaks.
	a := []int{3, 8, 2, 7, 9, 4, 1, 6, 5, 0, 10, 11}
	fmt.Printf("(a) a = %v\n", a)
	fmt.Printf("    is a peak:      %s   <- not monotone: no single boundary\n",
		bitString(len(a), func(i int) bool { return isPeak(a, i) }))
	fmt.Printf("    a[i] > a[i+1]:  %s    <- not monotone either\n",
		bitString(len(a)-1, func(i int) bool { return a[i] > a[i+1] }))
	lo, hi := 0, len(a)-1
	fmt.Printf("    start [%d, %d]  %s\n", lo, hi, invariant(a, lo, hi))
	for lo < hi {
		mid := int(uint(lo+hi) >> 1)
		if a[mid] < a[mid+1] {
			lo = mid + 1
			fmt.Printf("    mid=%2d  a[mid]=%2d < a[mid+1]=%2d  uphill   -> [%d, %d]  %s\n",
				mid, a[mid], a[mid+1], lo, hi, invariant(a, lo, hi))
		} else {
			hi = mid
			fmt.Printf("    mid=%2d  a[mid]=%2d > a[mid+1]=%2d  downhill -> [%d, %d]  %s\n",
				mid, a[mid], a[mid+1], lo, hi, invariant(a, lo, hi))
		}
	}
	argmax := slices.Index(a, slices.Max(a))
	fmt.Printf("    peak at %d (value %d); the global max is at %d (value %d)\n",
		lo, a[lo], argmax, a[argmax])

	// (b) Random distinct values: 10,000 permutations of 0..n-1, n in [1, 50].
	rng := rand.New(rand.NewPCG(1, 2))
	const trials = 10_000
	peakOK, isMax, isFirst, inBounds := 0, 0, 0, 0
	binTotal, scanTotal, peaksTotal, maxComps, maxScan := 0, 0, 0, 0, 0
	for range trials {
		n := 1 + rng.IntN(50)
		p := make([]int, n)
		for i := range p {
			p[i] = i
		}
		for i := n - 1; i > 0; i-- { // Fisher-Yates with IntN (portable to Rust)
			j := rng.IntN(i + 1)
			p[i], p[j] = p[j], p[i]
		}
		idx, comps := findPeak(p)
		// The oracle: a linear scan over the definition.
		first, peaks := -1, 0
		for i := range p {
			if isPeak(p, i) {
				peaks++
				if first < 0 {
					first = i
				}
			}
		}
		// What a linear peak search pays: first i with p[i] > p[i+1].
		scan := 0
		for i := 0; i < n-1; i++ {
			scan++
			if p[i] > p[i+1] {
				break
			}
		}
		if isPeak(p, idx) {
			peakOK++
		}
		if p[idx] == n-1 {
			isMax++
		}
		if idx == first {
			isFirst++
		}
		if comps >= bits.Len(uint(n))-1 && comps <= bits.Len(uint(n-1)) { // [floor, ceil] of log2 n
			inBounds++
		}
		binTotal += comps
		scanTotal += scan
		peaksTotal += peaks
		maxComps = max(maxComps, comps)
		maxScan = max(maxScan, scan)
	}
	fmt.Println()
	fmt.Printf("(b) %d random permutations, n in [1, 50], PCG(1, 2)\n", trials)
	fmt.Printf("    returned index is a peak (checked by scan):  %d / %d\n", peakOK, trials)
	fmt.Printf("    ... and it is the global maximum:            %d / %d (%.1f%%)\n",
		isMax, trials, 100*float64(isMax)/trials)
	fmt.Printf("    ... and it is the leftmost peak:             %d / %d\n", isFirst, trials)
	fmt.Printf("    avg peaks per array: %.2f -- binary search finds A peak, not THE max\n",
		float64(peaksTotal)/trials)
	fmt.Printf("    comparisons within [floor, ceil] of log2 n:  %d / %d (max seen %d, ceil(log2 50) = %d)\n",
		inBounds, trials, maxComps, bits.Len(49))
	fmt.Printf("    comparisons, total / worst: binary %d / %d, left-to-right scan %d / %d\n",
		binTotal, maxComps, scanTotal, maxScan)
	if scanTotal < binTotal {
		fmt.Printf("    -> the scan is %.1fx cheaper on average here: a random array falls early.\n",
			float64(binTotal)/float64(scanTotal))
	} else {
		fmt.Printf("    -> the binary search is %.1fx cheaper on average here.\n",
			float64(scanTotal)/float64(binTotal))
	}
	up := make([]int, 50) // the scan's worst case: no descent until the end
	for i := range up {
		up[i] = i
	}
	_, upComps := findPeak(up)
	fmt.Printf("       The guarantee is the worst case: on ascending 0..49 the scan needs %d, binary %d.\n",
		len(up)-1, upComps)

	// (c) A mountain (bitonic) array: strictly up to the peak, strictly down
	// after it. Here the only peak IS the maximum.
	const n, top = 1_000_000, 618_033
	m := make([]int, n)
	for i := range m {
		if i <= top {
			m[i] = 3 * i // up in steps of 3
		} else {
			m[i] = 3*top - 2*(i-top) // down in steps of 2: values repeat across sides
		}
	}
	peak, comps := findPeak(m)
	calls := 0
	viaSearch := sort.Search(n-1, func(i int) bool { calls++; return m[i] > m[i+1] })
	fmt.Println()
	fmt.Printf("(c) mountain array, n = %d, built with its top at %d\n", n, top)
	fmt.Printf("    findPeak: index %d in %d comparisons (a scan for the max: %d)\n", peak, comps, n-1)
	fmt.Printf("    stdlib cross-check: sort.Search(n-1, m[i] > m[i+1]) = %d (%d predicate calls)\n",
		viaSearch, calls)
	fmt.Println("    find t: ascending half with >=, descending half with <= (sort.Search's doc)")
	fmt.Printf("    %8s  %-17s %-17s %-17s %s\n", "t", "left (reads)", "right (reads)", "scan says", "stdlib agrees")
	worst := 0
	for _, t := range []int{300_000, 1_500_003, 1_200_001, 3 * top, 1000} {
		reads := 0
		var got []int
		// Ascending half m[0..peak]: first i with m[i] >= t.
		i := sort.Search(peak+1, func(i int) bool { reads++; return m[i] >= t })
		left := "-"
		if i <= peak {
			reads++
			if m[i] == t {
				got = append(got, i)
				left = fmt.Sprint(i)
			}
		}
		left = fmt.Sprintf("%s (%d)", left, reads)
		// Descending half m[peak+1..n): first j with m[j] <= t.
		total := comps + reads
		reads = 0
		j := peak + 1 + sort.Search(n-peak-1, func(k int) bool { reads++; return m[peak+1+k] <= t })
		right := "-"
		if j < n {
			reads++
			if m[j] == t {
				got = append(got, j)
				right = fmt.Sprint(j)
			}
		}
		right = fmt.Sprintf("%s (%d)", right, reads)
		worst = max(worst, total+reads)
		var scan []int // the oracle: every index holding t
		for k, v := range m {
			if v == t {
				scan = append(scan, k)
			}
		}
		// Stdlib: slices.BinarySearch for the ascending half; for the
		// descending half, BinarySearchFunc with the comparison reversed.
		_, okL := slices.BinarySearch(m[:peak+1], t)
		_, okR := slices.BinarySearchFunc(m[peak+1:], t, func(e, t int) int { return cmp.Compare(t, e) })
		agrees := okL == (len(got) > 0 && got[0] <= peak) && okR == (len(got) > 0 && got[len(got)-1] > peak)
		match := "MISMATCH"
		if slices.Equal(got, scan) {
			match = fmt.Sprint(scan)
		}
		fmt.Printf("    %8d  %-17s %-17s %-17s %v\n", t, left, right, match, agrees)
	}
	fmt.Printf("    whole mountain search (peak + both halves): at most %d reads here, a scan reads %d\n",
		worst, n)
}
```

**Output:**

```
(a) a = [3 8 2 7 9 4 1 6 5 0 10 11]
    is a peak:      010010010001   <- not monotone: no single boundary
    a[i] > a[i+1]:  01001101100    <- not monotone either
    start [0, 11]  invariant holds
    mid= 5  a[mid]= 4 > a[mid+1]= 1  downhill -> [0, 5]  invariant holds
    mid= 2  a[mid]= 2 < a[mid+1]= 7  uphill   -> [3, 5]  invariant holds
    mid= 4  a[mid]= 9 > a[mid+1]= 4  downhill -> [3, 4]  invariant holds
    mid= 3  a[mid]= 7 < a[mid+1]= 9  uphill   -> [4, 4]  invariant holds
    peak at 4 (value 9); the global max is at 11 (value 11)

(b) 10000 random permutations, n in [1, 50], PCG(1, 2)
    returned index is a peak (checked by scan):  10000 / 10000
    ... and it is the global maximum:            2425 / 10000 (24.2%)
    ... and it is the leftmost peak:             1660 / 10000
    avg peaks per array: 8.74 -- binary search finds A peak, not THE max
    comparisons within [floor, ceil] of log2 n:  10000 / 10000 (max seen 6, ceil(log2 50) = 6)
    comparisons, total / worst: binary 41322 / 6, left-to-right scan 16685 / 7
    -> the scan is 2.5x cheaper on average here: a random array falls early.
       The guarantee is the worst case: on ascending 0..49 the scan needs 49, binary 5.

(c) mountain array, n = 1000000, built with its top at 618033
    findPeak: index 618033 in 20 comparisons (a scan for the max: 999999)
    stdlib cross-check: sort.Search(n-1, m[i] > m[i+1]) = 618033 (20 predicate calls)
    find t: ascending half with >=, descending half with <= (sort.Search's doc)
           t  left (reads)      right (reads)     scan says         stdlib agrees
      300000  100000 (20)       - (18)            [100000]          true
     1500003  500001 (20)       795081 (20)       [500001 795081]   true
     1200001  - (20)            945082 (20)       [945082]          true
     1854099  618033 (20)       - (20)            [618033]          true
        1000  - (20)            - (18)            []                true
    whole mountain search (peak + both halves): at most 60 reads here, a scan reads 1000000
```

**Rust:**

```rust
// Binary search does not need sorted data. It needs a rule that says which
// half still holds an answer. For a PEAK (an element larger than its
// neighbours; outside the array counts as -infinity):
//
//   a[mid] < a[mid+1]  -> walking uphill to the right must top out somewhere
//                         right of mid: a peak lies in [mid+1, hi]
//   otherwise          -> a[mid] > a[mid+1] (distinct values): a peak lies
//                         in [lo, mid]
//
// Invariant on the window [lo, hi]:
//   left end climbs:   lo == 0   || a[lo-1] < a[lo]
//   right end falls:   hi == n-1 || a[hi]   > a[hi+1]
// A window with a rising left end and a falling right end contains a peak, and
// when it shrinks to one element, that element IS a peak.

/// A port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and its IntN, so
/// that seed (1, 2) produces exactly the permutations the Go program tests.
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

/// Returns a peak index and the number of a[mid] vs a[mid+1] comparisons.
/// Window [lo, hi] closed, so mid+1 <= hi is always in range. `None` only for
/// an empty slice: Go's version gets hi = -1 there and returns index 0, which
/// is out of range; in Rust `len() - 1` has no usize value, so ask checked_sub.
fn find_peak(a: &[i64]) -> Option<(usize, usize)> {
    let (mut lo, mut hi, mut comps) = (0, a.len().checked_sub(1)?, 0);
    while lo < hi {
        let mid = lo.midpoint(hi);
        comps += 1;
        if a[mid] < a[mid + 1] {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    Some((lo, comps))
}

fn is_peak(a: &[i64], i: usize) -> bool {
    (i == 0 || a[i - 1] < a[i]) && (i == a.len() - 1 || a[i] > a[i + 1])
}

fn invariant(a: &[i64], lo: usize, hi: usize) -> &'static str {
    if (lo == 0 || a[lo - 1] < a[lo]) && (hi == a.len() - 1 || a[hi] > a[hi + 1]) {
        "invariant holds"
    } else {
        "INVARIANT BROKEN"
    }
}

/// A predicate over 0..n printed as 0/1.
fn bit_string(n: usize, p: impl Fn(usize) -> bool) -> String {
    (0..n).map(|i| if p(i) { '1' } else { '0' }).collect()
}

/// Go's sort.Search: the first i in [0, n) with f(i) true, or n. Rust's std
/// has no search over an integer range, so this is the 8-line version (same
/// loop as Go's, so the same number of f calls).
fn search(n: usize, mut f: impl FnMut(usize) -> bool) -> usize {
    let (mut lo, mut hi) = (0, n);
    while lo < hi {
        let mid = lo.midpoint(hi);
        if f(mid) {
            hi = mid;
        } else {
            lo = mid + 1;
        }
    }
    lo
}

/// Go's %v on a slice of ints: `[1 2 3]`.
fn go_list(xs: &[usize]) -> String {
    let cells: Vec<String> = xs.iter().map(usize::to_string).collect();
    format!("[{}]", cells.join(" "))
}

fn main() {
    // (a) A trace on unsorted data with four peaks.
    let a: [i64; 12] = [3, 8, 2, 7, 9, 4, 1, 6, 5, 0, 10, 11];
    let shown: Vec<String> = a.iter().map(i64::to_string).collect();
    println!("(a) a = [{}]", shown.join(" "));
    println!(
        "    is a peak:      {}   <- not monotone: no single boundary",
        bit_string(a.len(), |i| is_peak(&a, i))
    );
    println!(
        "    a[i] > a[i+1]:  {}    <- not monotone either",
        bit_string(a.len() - 1, |i| a[i] > a[i + 1])
    );
    let (mut lo, mut hi) = (0, a.len() - 1);
    println!("    start [{lo}, {hi}]  {}", invariant(&a, lo, hi));
    while lo < hi {
        let mid = lo.midpoint(hi);
        let (x, y) = (a[mid], a[mid + 1]);
        if x < y {
            lo = mid + 1;
            println!(
                "    mid={mid:2}  a[mid]={x:2} < a[mid+1]={y:2}  uphill   -> [{lo}, {hi}]  {}",
                invariant(&a, lo, hi)
            );
        } else {
            hi = mid;
            println!(
                "    mid={mid:2}  a[mid]={x:2} > a[mid+1]={y:2}  downhill -> [{lo}, {hi}]  {}",
                invariant(&a, lo, hi)
            );
        }
    }
    // max_by_key returns the LAST maximum on ties; values are distinct here.
    let (argmax, max) = a.iter().enumerate().max_by_key(|&(_, v)| v).unwrap();
    println!(
        "    peak at {lo} (value {}); the global max is at {argmax} (value {max})",
        a[lo]
    );

    // (b) Random distinct values: 10,000 permutations of 0..n-1, n in [1, 50].
    let mut rng = Pcg::new(1, 2);
    const TRIALS: u32 = 10_000;
    let (mut peak_ok, mut is_max, mut is_first, mut in_bounds) = (0, 0, 0, 0);
    let (mut bin_total, mut scan_total, mut peaks_total) = (0, 0, 0);
    let (mut max_comps, mut max_scan) = (0, 0);
    for _ in 0..TRIALS {
        let n = 1 + rng.int_n(50);
        let mut p: Vec<i64> = (0..n as i64).collect();
        for i in (1..n).rev() {
            // Fisher-Yates with IntN, as in Go. (`p.swap`, not a tuple swap.)
            let j = rng.int_n(i + 1);
            p.swap(i, j);
        }
        let (idx, comps) = find_peak(&p).expect("n >= 1");
        // The oracle: a linear scan over the definition.
        let peaks: Vec<usize> = (0..n).filter(|&i| is_peak(&p, i)).collect();
        // What a linear peak search pays: first i with p[i] > p[i+1].
        let scan = match p.windows(2).position(|w| w[0] > w[1]) {
            Some(i) => i + 1, // comparisons up to and including the descent
            None => n - 1,    // climbed all the way: every pair compared
        };
        if is_peak(&p, idx) {
            peak_ok += 1;
        }
        if p[idx] == n as i64 - 1 {
            is_max += 1;
        }
        if peaks.first() == Some(&idx) {
            is_first += 1;
        }
        // [floor, ceil] of log2 n: n.ilog2() and Go's bits.Len(n-1)
        let ceil = (usize::BITS - (n - 1).leading_zeros()) as usize;
        if comps >= n.ilog2() as usize && comps <= ceil {
            in_bounds += 1;
        }
        bin_total += comps;
        scan_total += scan;
        peaks_total += peaks.len();
        max_comps = max_comps.max(comps);
        max_scan = max_scan.max(scan);
    }
    println!();
    println!("(b) {TRIALS} random permutations, n in [1, 50], PCG(1, 2)");
    println!("    returned index is a peak (checked by scan):  {peak_ok} / {TRIALS}");
    println!(
        "    ... and it is the global maximum:            {is_max} / {TRIALS} ({:.1}%)",
        100.0 * f64::from(is_max) / f64::from(TRIALS)
    );
    println!("    ... and it is the leftmost peak:             {is_first} / {TRIALS}");
    println!(
        "    avg peaks per array: {:.2} -- binary search finds A peak, not THE max",
        peaks_total as f64 / f64::from(TRIALS)
    );
    println!(
        "    comparisons within [floor, ceil] of log2 n:  {in_bounds} / {TRIALS} (max seen {max_comps}, ceil(log2 50) = {})",
        usize::BITS - 49usize.leading_zeros()
    );
    println!(
        "    comparisons, total / worst: binary {bin_total} / {max_comps}, left-to-right scan {scan_total} / {max_scan}"
    );
    if scan_total < bin_total {
        println!(
            "    -> the scan is {:.1}x cheaper on average here: a random array falls early.",
            bin_total as f64 / scan_total as f64
        );
    } else {
        println!(
            "    -> the binary search is {:.1}x cheaper on average here.",
            scan_total as f64 / bin_total as f64
        );
    }
    let up: Vec<i64> = (0..50).collect(); // the scan's worst case: no descent until the end
    let (_, up_comps) = find_peak(&up).expect("not empty");
    println!(
        "       The guarantee is the worst case: on ascending 0..49 the scan needs {}, binary {up_comps}.",
        up.len() - 1
    );

    // (c) A mountain (bitonic) array: strictly up to the peak, strictly down
    // after it. Here the only peak IS the maximum.
    const N: usize = 1_000_000;
    const TOP: i64 = 618_033;
    let m: Vec<i64> = (0..N as i64)
        .map(|i| {
            if i <= TOP {
                3 * i // up in steps of 3
            } else {
                3 * TOP - 2 * (i - TOP) // down in steps of 2: values repeat across sides
            }
        })
        .collect();
    let (peak, comps) = find_peak(&m).expect("not empty");
    // Cross-check: on a mountain the descent test "m[i] > m[i+1]" is a monotone
    // false...false true...true predicate over the n-1 adjacent pairs, so the
    // first descent is a plain range search. (The tempting std one-liner
    // `m.windows(2).partition_point(..)` does not compile: `Windows` is an
    // iterator, and `partition_point` exists only on slices.)
    let mut calls = 0;
    let via_search = search(N - 1, |i| {
        calls += 1;
        m[i] > m[i + 1]
    });
    println!();
    println!("(c) mountain array, n = {N}, built with its top at {TOP}");
    println!(
        "    findPeak: index {peak} in {comps} comparisons (a scan for the max: {})",
        N - 1
    );
    println!(
        "    cross-check: search(N - 1, |i| m[i] > m[i + 1]) = {via_search} ({calls} predicate calls)"
    );
    println!("    find t: ascending half with >=, descending half with <= (sort.Search's doc)");
    println!(
        "    {:>8}  {:<17} {:<17} {:<17} stdlib agrees",
        "t", "left (reads)", "right (reads)", "scan says"
    );
    let mut worst = 0;
    for t in [300_000, 1_500_003, 1_200_001, 3 * TOP, 1000] {
        let mut reads = 0;
        let mut got = Vec::new();
        // Ascending half m[..=peak]: first i with m[i] >= t.
        let i = search(peak + 1, |i| {
            reads += 1;
            m[i] >= t
        });
        let mut left = "-".to_string();
        if i <= peak {
            reads += 1;
            if m[i] == t {
                got.push(i);
                left = i.to_string();
            }
        }
        let left = format!("{left} ({reads})");
        let total = comps + reads;
        // Descending half m[peak+1..]: first j with m[j] <= t.
        let mut reads = 0;
        let j = peak
            + 1
            + search(N - peak - 1, |k| {
                reads += 1;
                m[peak + 1 + k] <= t
            });
        let mut right = "-".to_string();
        if j < N {
            reads += 1;
            if m[j] == t {
                got.push(j);
                right = j.to_string();
            }
        }
        let right = format!("{right} ({reads})");
        worst = worst.max(total + reads);
        // The oracle: every index holding t.
        let scan: Vec<usize> = (0..N).filter(|&k| m[k] == t).collect();
        // Std: partition_point on each half as a slice. It returns the first
        // index where the predicate is FALSE, so the ascending half asks
        // `v < t` and the descending half flips it to `v > t`.
        let (asc, desc) = (&m[..=peak], &m[peak + 1..]);
        let pl = asc.partition_point(|&v| v < t);
        let pr = desc.partition_point(|&v| v > t);
        let ok_l = asc.get(pl) == Some(&t);
        let ok_r = desc.get(pr) == Some(&t);
        let agrees = ok_l == got.first().is_some_and(|&g| g <= peak)
            && ok_r == got.last().is_some_and(|&g| g > peak);
        let matched = if got == scan {
            go_list(&scan)
        } else {
            "MISMATCH".to_string()
        };
        println!("    {t:>8}  {left:<17} {right:<17} {matched:<17} {agrees}");
    }
    println!(
        "    whole mountain search (peak + both halves): at most {worst} reads here, a scan reads {N}"
    );
}
```

**Rust output:**

```
(a) a = [3 8 2 7 9 4 1 6 5 0 10 11]
    is a peak:      010010010001   <- not monotone: no single boundary
    a[i] > a[i+1]:  01001101100    <- not monotone either
    start [0, 11]  invariant holds
    mid= 5  a[mid]= 4 > a[mid+1]= 1  downhill -> [0, 5]  invariant holds
    mid= 2  a[mid]= 2 < a[mid+1]= 7  uphill   -> [3, 5]  invariant holds
    mid= 4  a[mid]= 9 > a[mid+1]= 4  downhill -> [3, 4]  invariant holds
    mid= 3  a[mid]= 7 < a[mid+1]= 9  uphill   -> [4, 4]  invariant holds
    peak at 4 (value 9); the global max is at 11 (value 11)

(b) 10000 random permutations, n in [1, 50], PCG(1, 2)
    returned index is a peak (checked by scan):  10000 / 10000
    ... and it is the global maximum:            2425 / 10000 (24.2%)
    ... and it is the leftmost peak:             1660 / 10000
    avg peaks per array: 8.74 -- binary search finds A peak, not THE max
    comparisons within [floor, ceil] of log2 n:  10000 / 10000 (max seen 6, ceil(log2 50) = 6)
    comparisons, total / worst: binary 41322 / 6, left-to-right scan 16685 / 7
    -> the scan is 2.5x cheaper on average here: a random array falls early.
       The guarantee is the worst case: on ascending 0..49 the scan needs 49, binary 5.

(c) mountain array, n = 1000000, built with its top at 618033
    findPeak: index 618033 in 20 comparisons (a scan for the max: 999999)
    cross-check: search(N - 1, |i| m[i] > m[i + 1]) = 618033 (20 predicate calls)
    find t: ascending half with >=, descending half with <= (sort.Search's doc)
           t  left (reads)      right (reads)     scan says         stdlib agrees
      300000  100000 (20)       - (18)            [100000]          true
     1500003  500001 (20)       795081 (20)       [500001 795081]   true
     1200001  - (20)            945082 (20)       [945082]          true
     1854099  618033 (20)       - (20)            [618033]          true
        1000  - (20)            - (18)            []                true
    whole mountain search (peak + both halves): at most 60 reads here, a scan reads 1000000
```

> **Go vs Rust:** Same output as Go except the label of one line, diffed: the PCG-DXSM + `IntN` port feeds both programs the same 10,000 permutations, and every count matches (41,322 binary comparisons, 2,425 maxima, the 20/18 reads per half). Rust's std has no search over an integer range, so `search(n, f)` is a hand-written 8-line copy of `sort.Search`. The tempting one-liner `m.windows(2).partition_point(|w| w[0] < w[1])` fails to compile (E0599: `Windows` is an iterator, and `partition_point` exists only on slices). On slices, `partition_point` returns the first index where the predicate is **false**, so the stdlib check asks `v < t` on the ascending half and `v > t` on the descending one. `find_peak` returns `Option` because `len() - 1` has no `usize` value for an empty slice. Go's version gets `hi = -1` there and returns the out-of-range index 0.

**Complexity:** Θ(log n) comparisons on unsorted input, within [⌊log₂ n⌋, ⌈log₂ n⌉] in all 10,000 random arrays (max 6 at n ≤ 50) · it finds *a* peak: the global maximum in only 2,425 of 10,000 (the arrays average 8.74 peaks) · on random data the scan was 2.5× cheaper on average (16,685 vs 41,322 comparisons); the guarantee is the worst case (ascending input: 49 vs 5) · mountain search Θ(log n) total: 20 comparisons for the top, at most 60 reads for the whole search, against 10⁶ for a scan.

---

> Next tier: [🔴 hard](3-hard.md) — every way to get the boundaries wrong, run; floats and their bits;
> faster layouts; galloping; and the capstone.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
