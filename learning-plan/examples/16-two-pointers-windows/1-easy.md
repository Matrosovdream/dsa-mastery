# Step 16 — Two Pointers, Sliding Window & Prefix Sums · 🟢 Easy

Examples **1–6**: pair sum with two pointers walking inward, an in-place dedupe that keeps at most k
copies, a fixed window that adds one element and drops one, and the prefix-sum family — a range sum in
Θ(1), the overflow and underflow hazards of the subtraction, and difference arrays.

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

> ← Back to the [index](README.md) · Progress: [PROGRESS.md](PROGRESS.md) · Next tier: [🟡 medium](2-medium.md)

---

## 1. Pair sum: two pointers walking inward

`🟢 easy` · *Two pointers, converging*

On a sorted array, one comparison of `a[lo] + a[hi]` with the target discards a whole row or column of the n×n pair matrix: if the sum is too small, `a[lo]` cannot pair with anything left in `[lo+1, hi]`, so `lo++` retires `hi − lo` pairs in one step (and symmetrically for `hi--`). The invariant — *if any pair sums to the target, both indices lie in `[lo, hi]`* — makes it correct, and each step shrinks `hi − lo` by one, so the loop runs at most n − 1 times: Θ(n) against the n(n−1)/2 pairs of brute force, for the price of sorted input.

**Steps:**

1. Trace the loop on a small sorted array, counting the pairs each step retires against the pairs still inside the window.
2. Take the worst case (no pair exists) at four sizes, and watch the step count stay exactly n − 1.
3. Check the loop against brute force on 20,000 random cases, then break `lo < hi` into `lo <= hi` and count what goes wrong.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
)

// pairSum looks for i < j with a[i]+a[j] == target in a SORTED slice.
//
// Invariant: if any pair sums to target, both indices lie in [lo, hi].
// Every step shrinks hi-lo by one, so there are at most len(a)-1 steps.
// It returns the number of loop iterations ("steps") as well.
func pairSum(a []int, target int) (i, j, steps int, ok bool) {
	lo, hi := 0, len(a)-1 // for an empty slice hi is -1 and the loop never runs
	for lo < hi {         // strict: lo == hi would pair an element with itself
		steps++
		switch sum := a[lo] + a[hi]; {
		case sum == target:
			return lo, hi, steps, true
		case sum < target:
			lo++ // a[lo] is too small to pair with anything in (lo, hi]
		default:
			hi-- // a[hi] is too big to pair with anything in [lo, hi)
		}
	}
	return -1, -1, steps, false
}

// pairSumBuggy is the same loop with ONE character changed: <= instead of <.
func pairSumBuggy(a []int, target int) (i, j int, ok bool) {
	lo, hi := 0, len(a)-1
	for lo <= hi { // BUG: when lo == hi, a[lo] is paired with itself
		switch sum := a[lo] + a[hi]; {
		case sum == target:
			return lo, hi, true
		case sum < target:
			lo++
		default:
			hi--
		}
	}
	return -1, -1, false
}

// bruteForce tries every pair and counts how many it tried.
func bruteForce(a []int, target int) (tried int, ok bool) {
	for i := range a {
		for j := i + 1; j < len(a); j++ {
			tried++
			if a[i]+a[j] == target {
				return tried, true
			}
		}
	}
	return tried, false
}

// choose2 is C(m, 2): the number of pairs among m elements.
func choose2(m int) int { return m * (m - 1) / 2 }

// trace prints one row per step: the window and values BEFORE the step, the
// action, then the books AFTER it. "retired" is accumulated step by step (a
// step retires hi-lo pairs), "left" is C(window size, 2); the two must always
// add up to all n(n-1)/2 pairs.
func trace(a []int, target int) {
	n := len(a)
	fmt.Printf("a = %v, target %d\n", a, target)
	fmt.Println("  step  lo hi  a[lo] a[hi]   sum  action   retired  left")
	lo, hi, retired, held, step := 0, n-1, 0, 0, 0
	for lo < hi {
		step++
		l, h, sum := lo, hi, a[lo]+a[hi]
		action := "found"
		switch {
		case sum < target:
			action, retired, lo = "lo++", retired+(hi-lo), lo+1
		case sum > target:
			action, retired, hi = "hi--", retired+(hi-lo), hi-1
		}
		left := choose2(hi - lo + 1)
		fmt.Printf("  %4d  %2d %2d  %5d %5d  %4d  %-7s  %7d  %4d\n",
			step, l, h, a[l], a[h], sum, action, retired, left)
		if retired+left == choose2(n) {
			held++
		}
		if action == "found" {
			break
		}
	}
	fmt.Printf("  retired + left == n(n-1)/2 == %d held at %d of %d steps\n", choose2(n), held, step)
}

func main() {
	fmt.Println("== (a) the trace ==")
	a := []int{1, 3, 4, 6, 8, 11, 15}
	trace(a, 14)
	trace(a, 20)

	fmt.Println()
	fmt.Println("== (b) worst case: no pair reaches the target ==")
	fmt.Printf("  %6s %12s %14s %10s\n", "n", "steps", "brute pairs", "ratio")
	for _, n := range []int{10, 100, 1000, 10000} {
		xs := make([]int, n)
		for i := range xs {
			xs[i] = i
		}
		_, _, steps, _ := pairSum(xs, 4*n) // bigger than any a[i]+a[j]
		tried, _ := bruteForce(xs, 4*n)
		fmt.Printf("  %6d %12d %14d %9.1fx   (n-1 = %d, n/2 = %.1f)\n",
			n, steps, tried, float64(tried)/float64(steps), n-1, float64(n)/2)
	}

	fmt.Println()
	fmt.Println("== (c) 20000 random cases against brute force ==")
	rng := rand.New(rand.NewPCG(1, 2))
	const cases = 20000
	var wrong, noPair, buggyWrong, selfPairs int
	var first string
	for range cases {
		a := make([]int, rng.IntN(13))
		for i := range a {
			a[i] = rng.IntN(10)
		}
		slices.Sort(a)
		target := rng.IntN(20)

		_, want := bruteForce(a, target)
		i, j, _, got := pairSum(a, target)
		if got != want || (got && !(i < j && a[i]+a[j] == target)) {
			wrong++
		}
		if !want {
			noPair++
		}
		bi, bj, bgot := pairSumBuggy(a, target)
		if bgot != want || (bgot && !(bi < bj && a[bi]+a[bj] == target)) {
			buggyWrong++
			if bgot && bi == bj {
				selfPairs++
			}
			if first == "" {
				first = fmt.Sprintf("a = %v, target %d: brute force says %v, lo <= hi returns (%d, %d, %v)",
					a, target, want, bi, bj, bgot)
			}
		}
	}
	fmt.Printf("  lo <  hi: %d wrong of %d cases (%d of the cases have no pair at all)\n", wrong, cases, noPair)
	fmt.Printf("  lo <= hi: %d wrong of %d cases (%d of them returned i == j)\n", buggyWrong, cases, selfPairs)
	fmt.Println("  first counterexample:", first)
	fmt.Println("  same family as lesson 10, example 12: an element that is allowed to pair with itself.")

	fmt.Println()
	fmt.Println("== (d) n = 0 and n = 1 ==")
	var nilSlice []int
	for _, a := range [][]int{nilSlice, {}, {7}} {
		i, j, steps, ok := pairSum(a, 14)
		fmt.Printf("  len %d (nil: %-5v): (%d, %d, %v) after %d steps\n", len(a), a == nil, i, j, ok, steps)
	}
}
```

**Output:**

```
== (a) the trace ==
a = [1 3 4 6 8 11 15], target 14
  step  lo hi  a[lo] a[hi]   sum  action   retired  left
     1   0  6      1    15    16  hi--           6    15
     2   0  5      1    11    12  lo++          11    10
     3   1  5      3    11    14  found         11    10
  retired + left == n(n-1)/2 == 21 held at 3 of 3 steps
a = [1 3 4 6 8 11 15], target 20
  step  lo hi  a[lo] a[hi]   sum  action   retired  left
     1   0  6      1    15    16  lo++           6    15
     2   1  6      3    15    18  lo++          11    10
     3   2  6      4    15    19  lo++          15     6
     4   3  6      6    15    21  hi--          18     3
     5   3  5      6    11    17  lo++          20     1
     6   4  5      8    11    19  lo++          21     0
  retired + left == n(n-1)/2 == 21 held at 6 of 6 steps

== (b) worst case: no pair reaches the target ==
       n        steps    brute pairs      ratio
      10            9             45       5.0x   (n-1 = 9, n/2 = 5.0)
     100           99           4950      50.0x   (n-1 = 99, n/2 = 50.0)
    1000          999         499500     500.0x   (n-1 = 999, n/2 = 500.0)
   10000         9999       49995000    5000.0x   (n-1 = 9999, n/2 = 5000.0)

== (c) 20000 random cases against brute force ==
  lo <  hi: 0 wrong of 20000 cases (12098 of the cases have no pair at all)
  lo <= hi: 1766 wrong of 20000 cases (1766 of them returned i == j)
  first counterexample: a = [1 2 4 5 5 8 9 9], target 8: brute force says false, lo <= hi returns (2, 2, true)
  same family as lesson 10, example 12: an element that is allowed to pair with itself.

== (d) n = 0 and n = 1 ==
  len 0 (nil: true ): (-1, -1, false) after 0 steps
  len 0 (nil: false): (-1, -1, false) after 0 steps
  len 1 (nil: false): (-1, -1, false) after 0 steps
```

**Rust:**

```rust
use std::cmp::Ordering;

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) feeds this program the same numbers that Go's
// rand.New(rand.NewPCG(1, 2)) feeds the Go program.
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

// pair_sum looks for i < j with a[i]+a[j] == target in a SORTED slice.
//
// Invariant: if any pair sums to target, both indices lie in [lo, hi].
// Every step shrinks hi-lo by one, so there are at most len-1 steps.
// It returns the number of loop iterations ("steps") as well.
fn pair_sum(a: &[i64], target: i64) -> (Option<(usize, usize)>, usize) {
    // Go's `len(a)-1` is -1 for an empty slice; `usize` has no -1, so say so.
    let Some(mut hi) = a.len().checked_sub(1) else {
        return (None, 0);
    };
    let mut lo = 0;
    let mut steps = 0;
    while lo < hi {
        // strict: lo == hi would pair an element with itself. It also
        // protects `hi -= 1` below: hi > lo >= 0, so it cannot underflow.
        steps += 1;
        match (a[lo] + a[hi]).cmp(&target) {
            Ordering::Equal => return (Some((lo, hi)), steps),
            Ordering::Less => lo += 1, // a[lo] is too small to pair with anything in (lo, hi]
            Ordering::Greater => hi -= 1, // a[hi] is too big to pair with anything in [lo, hi)
        }
    }
    (None, steps)
}

// pair_sum_buggy is the same loop with ONE character changed: <= instead of <.
fn pair_sum_buggy(a: &[i64], target: i64) -> Option<(usize, usize)> {
    let mut hi = a.len().checked_sub(1)?;
    let mut lo = 0;
    while lo <= hi {
        // BUG: when lo == hi, a[lo] is paired with itself
        match (a[lo] + a[hi]).cmp(&target) {
            Ordering::Equal => return Some((lo, hi)),
            Ordering::Less => lo += 1,
            // With `<=` the loop reaches lo == hi == 0, where `hi -= 1` would
            // underflow (panic in debug, wrap to usize::MAX under -O and then
            // index out of bounds). checked_sub is the Rust price of the bug;
            // Go just decrements to -1 and the loop test fails.
            Ordering::Greater => hi = hi.checked_sub(1)?,
        }
    }
    None
}

// brute_force tries every pair and counts how many it tried.
fn brute_force(a: &[i64], target: i64) -> (usize, bool) {
    let mut tried = 0;
    for (i, x) in a.iter().enumerate() {
        for y in &a[i + 1..] {
            tried += 1;
            if x + y == target {
                return (tried, true);
            }
        }
    }
    (tried, false)
}

// choose2 is C(m, 2): the number of pairs among m elements.
// (m - 1 would underflow for m == 0, hence saturating_sub.)
fn choose2(m: usize) -> usize {
    m * m.saturating_sub(1) / 2
}

// Go prints a slice as [1 3 4]; Rust's {:?} prints [1, 3, 4]. Format by hand.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

// trace prints one row per step: the window and values BEFORE the step, the
// action, then the books AFTER it. "retired" is accumulated step by step (a
// step retires hi-lo pairs), "left" is C(window size, 2); the two must always
// add up to all n(n-1)/2 pairs.
fn trace(a: &[i64], target: i64) {
    let n = a.len();
    println!("a = {}, target {target}", go_slice(a));
    println!("  step  lo hi  a[lo] a[hi]   sum  action   retired  left");
    let (mut lo, mut hi, mut retired, mut held, mut step) = (0, n - 1, 0, 0, 0);
    while lo < hi {
        step += 1;
        let (l, h, sum) = (lo, hi, a[lo] + a[hi]);
        let action = match sum.cmp(&target) {
            Ordering::Less => {
                retired += hi - lo;
                lo += 1;
                "lo++"
            }
            Ordering::Greater => {
                retired += hi - lo;
                hi -= 1;
                "hi--"
            }
            Ordering::Equal => "found",
        };
        let left = choose2(hi - lo + 1);
        println!(
            "  {step:>4}  {l:>2} {h:>2}  {:>5} {:>5}  {sum:>4}  {action:<7}  {retired:>7}  {left:>4}",
            a[l], a[h]
        );
        if retired + left == choose2(n) {
            held += 1;
        }
        if action == "found" {
            break;
        }
    }
    println!(
        "  retired + left == n(n-1)/2 == {} held at {held} of {step} steps",
        choose2(n)
    );
}

fn main() {
    println!("== (a) the trace ==");
    let a = [1, 3, 4, 6, 8, 11, 15];
    trace(&a, 14);
    trace(&a, 20);

    println!();
    println!("== (b) worst case: no pair reaches the target ==");
    println!(
        "  {:>6} {:>12} {:>14} {:>10}",
        "n", "steps", "brute pairs", "ratio"
    );
    for n in [10usize, 100, 1000, 10000] {
        let xs: Vec<i64> = (0..n as i64).collect();
        let target = 4 * n as i64; // bigger than any a[i]+a[j]
        let (_, steps) = pair_sum(&xs, target);
        let (tried, _) = brute_force(&xs, target);
        println!(
            "  {n:>6} {steps:>12} {tried:>14} {:>9.1}x   (n-1 = {}, n/2 = {:.1})",
            tried as f64 / steps as f64,
            n - 1,
            n as f64 / 2.0
        );
    }

    println!();
    println!("== (c) 20000 random cases against brute force ==");
    let mut rng = Pcg::new(1, 2);
    const CASES: usize = 20000;
    let (mut wrong, mut no_pair, mut buggy_wrong, mut self_pairs) = (0, 0, 0, 0);
    let mut first = String::new();
    for _ in 0..CASES {
        let len = rng.int_n(13) as usize;
        let mut a: Vec<i64> = (0..len).map(|_| rng.int_n(10) as i64).collect();
        a.sort();
        let target = rng.int_n(20) as i64;

        let (_, want) = brute_force(&a, target);
        let (found, _) = pair_sum(&a, target);
        let valid = |p: Option<(usize, usize)>| match p {
            Some((i, j)) => i < j && a[i] + a[j] == target,
            None => true,
        };
        if found.is_some() != want || !valid(found) {
            wrong += 1;
        }
        if !want {
            no_pair += 1;
        }
        let buggy = pair_sum_buggy(&a, target);
        if buggy.is_some() != want || !valid(buggy) {
            buggy_wrong += 1;
            if matches!(buggy, Some((i, j)) if i == j) {
                self_pairs += 1;
            }
            if first.is_empty() {
                first = format!(
                    "a = {}, target {target}: brute force says {want}, lo <= hi returns {buggy:?}",
                    go_slice(&a)
                );
            }
        }
    }
    println!(
        "  lo <  hi: {wrong} wrong of {CASES} cases ({no_pair} of the cases have no pair at all)"
    );
    println!(
        "  lo <= hi: {buggy_wrong} wrong of {CASES} cases ({self_pairs} of them returned i == j)"
    );
    println!("  first counterexample: {first}");
    println!(
        "  same family as lesson 10, example 12: an element that is allowed to pair with itself."
    );

    println!();
    println!("== (d) n = 0 and n = 1 ==");
    for a in [&[][..], &[7][..]] {
        let (found, steps) = pair_sum(a, 14);
        println!(
            "  len {}: len.checked_sub(1) = {:?}, pair_sum -> {found:?} after {steps} steps",
            a.len(),
            a.len().checked_sub(1)
        );
    }
}
```

**Rust output:**

```
== (a) the trace ==
a = [1 3 4 6 8 11 15], target 14
  step  lo hi  a[lo] a[hi]   sum  action   retired  left
     1   0  6      1    15    16  hi--           6    15
     2   0  5      1    11    12  lo++          11    10
     3   1  5      3    11    14  found         11    10
  retired + left == n(n-1)/2 == 21 held at 3 of 3 steps
a = [1 3 4 6 8 11 15], target 20
  step  lo hi  a[lo] a[hi]   sum  action   retired  left
     1   0  6      1    15    16  lo++           6    15
     2   1  6      3    15    18  lo++          11    10
     3   2  6      4    15    19  lo++          15     6
     4   3  6      6    15    21  hi--          18     3
     5   3  5      6    11    17  lo++          20     1
     6   4  5      8    11    19  lo++          21     0
  retired + left == n(n-1)/2 == 21 held at 6 of 6 steps

== (b) worst case: no pair reaches the target ==
       n        steps    brute pairs      ratio
      10            9             45       5.0x   (n-1 = 9, n/2 = 5.0)
     100           99           4950      50.0x   (n-1 = 99, n/2 = 50.0)
    1000          999         499500     500.0x   (n-1 = 999, n/2 = 500.0)
   10000         9999       49995000    5000.0x   (n-1 = 9999, n/2 = 5000.0)

== (c) 20000 random cases against brute force ==
  lo <  hi: 0 wrong of 20000 cases (12098 of the cases have no pair at all)
  lo <= hi: 1766 wrong of 20000 cases (1766 of them returned i == j)
  first counterexample: a = [1 2 4 5 5 8 9 9], target 8: brute force says false, lo <= hi returns Some((2, 2))
  same family as lesson 10, example 12: an element that is allowed to pair with itself.

== (d) n = 0 and n = 1 ==
  len 0: len.checked_sub(1) = None, pair_sum -> None after 0 steps
  len 1: len.checked_sub(1) = Some(0), pair_sum -> None after 0 steps
```

> **Go vs Rust:** Every count matches Go (9/99/999/9,999 steps, 12,098 no-pair cases, 1,766 wrong answers from `lo <= hi`): the `Pcg` port gives both programs the same 20,000 inputs. Only the printed forms differ: Rust returns `Option<(usize, usize)>` where Go returns `(i, j, ok)`, and there is no nil slice in Rust. `usize` is the real difference: Go's `len(a)-1` is `-1` for an empty slice and the loop test fails quietly, Rust must `checked_sub(1)` it; and in the `<=` bug `hi -= 1` at `hi == 0` underflows (in a scratch run without `-O` it panicked "attempt to subtract with overflow"; with `-O` it wrapped and panicked "index out of bounds ... 18446744073709551615"), so the buggy twin needs `checked_sub` just to be runnable.

**Complexity:** Θ(n) time after sorting, Θ(1) space. At most n − 1 steps, and exactly n − 1 when no pair exists (9, 99, 999 and 9,999 above), against n(n−1)/2 brute-force pairs: the ratio is n/2. Changing `lo < hi` to `lo <= hi` gave 1,766 wrong answers in 20,000 cases, and every one was an element paired with itself (`i == j`). Unsorted input costs a Θ(n log n) sort first (or lesson 10's hash map: Θ(n) time and space).

---

## 2. Dedupe in place, keeping at most k

`🟢 easy` · *Two pointers, read/write*

Same-direction pointers: `read` visits every element once and `write` marks the end of the answer built so far, so the invariant is *`a[:write]` is the answer for `a[:read]`*. To keep at most k copies of each value in a sorted slice, compare `a[read]` with `a[write-k]` — k places back in the **output**, not in the input — and keep it only if they differ; k = 1 is plain dedupe. That is Θ(n) time and Θ(1) extra space, but only because sorted input puts equal values next to each other.

**Steps:**

1. Write the loop and trace it on a small sorted array for k = 1 and k = 2, checking the invariant after every step against a from-scratch answer.
2. Count reads, writes and allocations at n = 10,000 and 1,000,000, then show what unsorted input does to it.
3. Check 20,000 random arrays against a counting oracle (and `slices.Compact` for k = 1), and print the backing array to see what is left beyond `write`.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
	"testing"
)

// dedupeAtMost keeps at most k copies of each value, IN PLACE, in a sorted
// slice, and returns the new length. It also counts element reads and writes.
//
// Invariant: a[:write] is the answer for a[:read]. The test looks k places
// back in the OUTPUT (a[write-k]), not the input: if the last k kept values
// all equal a[read], then keeping it would be copy number k+1.
func dedupeAtMost(a []int, k int) (write, reads, writes int) {
	if k < 1 { // "at most 0 copies" keeps nothing
		return 0, 0, 0
	}
	for read := range a {
		reads++
		if write < k || a[read] != a[write-k] { // write < k guards a[write-k]
			a[write] = a[read]
			write++
			writes++
		}
	}
	return write, reads, writes
}

// keepFirstK is the oracle: no pointers, a map of counts. It builds a new slice.
func keepFirstK(a []int, k int) []int {
	seen := map[int]int{}
	out := []int{}
	for _, v := range a {
		if seen[v] < k {
			out = append(out, v)
		}
		seen[v]++
	}
	return out
}

// trace is the same loop with a print per step. After every step it checks
// the invariant: a[:write] must equal the from-scratch answer for a0[:read+1].
func trace(a0 []int, k int) {
	a := slices.Clone(a0)
	fmt.Printf("k = %d, a = %v\n", k, a0)
	fmt.Println("  read write  a[read] a[write-k]  keep?  a[:write] after the step")
	write, held := 0, 0
	for read, v := range a {
		back, keep := "-", true
		if write >= k {
			back, keep = fmt.Sprint(a[write-k]), v != a[write-k]
		}
		w := write
		if keep {
			a[write] = v
			write++
		}
		if slices.Equal(a[:write], keepFirstK(a0[:read+1], k)) {
			held++
		}
		fmt.Printf("  %4d %5d  %6d %9s  %-5v  %v\n", read, w, v, back, keep, a[:write])
	}
	n, _, _ := dedupeAtMost(slices.Clone(a0), k)
	fmt.Printf("  invariant held at %d of %d steps; final length %d (dedupeAtMost says %d)\n", held, len(a0), write, n)
}

func main() {
	fmt.Println("== (a) the trace ==")
	a0 := []int{1, 1, 1, 2, 2, 3, 3, 3, 3, 4}
	trace(a0, 1)
	trace(a0, 2)

	fmt.Println()
	fmt.Println("== (b) counts on sorted random data (values IntN(n/4)) ==")
	rng := rand.New(rand.NewPCG(1, 2))
	fmt.Printf("  %9s %2s %9s %9s %9s %8s  %s\n", "n", "k", "reads", "writes", "kept", "allocs", "matches oracle")
	for _, n := range []int{10_000, 1_000_000} {
		src := make([]int, n)
		for i := range src {
			src[i] = rng.IntN(n / 4)
		}
		slices.Sort(src)
		buf := make([]int, n)
		for k := 1; k <= 3; k++ {
			var kept, reads, writes int
			allocs := testing.AllocsPerRun(3, func() {
				copy(buf, src) // restore the input; copy does not allocate
				kept, reads, writes = dedupeAtMost(buf, k)
			})
			fmt.Printf("  %9d %2d %9d %9d %9d %8.0f  %v\n", n, k, reads, writes, kept, allocs,
				slices.Equal(buf[:kept], keepFirstK(src, k)))
		}
	}

	fmt.Println()
	fmt.Println("== (c) the unsorted trap ==")
	u := []int{1, 2, 1, 2, 1}
	n, _, _ := dedupeAtMost(u, 1)
	fmt.Printf("  unsorted [1 2 1 2 1], k = 1: length %d, slice now %v (no two neighbours are equal)\n", n, u[:n])
	slices.Sort(u)
	n, _, _ = dedupeAtMost(u, 1)
	fmt.Printf("  sorted first:                length %d, slice now %v\n", n, u[:n])

	fmt.Println()
	fmt.Println("== (d) 20000 random sorted arrays (length <= 20, values IntN(6), k in 1..3) ==")
	var bad, ones, badCompact int
	for range 20000 {
		a := make([]int, rng.IntN(21))
		for i := range a {
			a[i] = rng.IntN(6)
		}
		slices.Sort(a)
		k := 1 + rng.IntN(3)
		want := keepFirstK(a, k)
		var compact []int
		if k == 1 {
			ones++
			compact = slices.Compact(slices.Clone(a))
		}
		n, _, _ := dedupeAtMost(a, k)
		if !slices.Equal(a[:n], want) {
			bad++
		}
		if k == 1 && !slices.Equal(a[:n], compact) {
			badCompact++
		}
	}
	fmt.Printf("  versus the counting oracle: %d mismatches; k = 1 versus slices.Compact: %d mismatches in %d cases\n",
		bad, badCompact, ones)

	fmt.Println()
	fmt.Println("== (e) what is left beyond write ==")
	a := slices.Clone(a0)
	n, _, _ = dedupeAtMost(a, 1)
	fmt.Printf("  by hand:        len %d, answer %v\n", n, a[:n])
	fmt.Printf("                  backing array %v  <- the tail is stale\n", a)
	b := slices.Clone(a0)
	c := slices.Compact(b)
	fmt.Printf("  slices.Compact: len %d, answer %v\n", len(c), c)
	fmt.Printf("                  backing array %v  <- the tail is zeroed\n", b)
	fmt.Println("  (read in the source, not run: Compact ends with clear(s[k:]) for that reason; with")
	fmt.Println("  pointer elements a stale tail would also keep the pointed-to values alive.)")
}
```

**Output:**

```
== (a) the trace ==
k = 1, a = [1 1 1 2 2 3 3 3 3 4]
  read write  a[read] a[write-k]  keep?  a[:write] after the step
     0     0       1         -  true   [1]
     1     1       1         1  false  [1]
     2     1       1         1  false  [1]
     3     1       2         1  true   [1 2]
     4     2       2         2  false  [1 2]
     5     2       3         2  true   [1 2 3]
     6     3       3         3  false  [1 2 3]
     7     3       3         3  false  [1 2 3]
     8     3       3         3  false  [1 2 3]
     9     3       4         3  true   [1 2 3 4]
  invariant held at 10 of 10 steps; final length 4 (dedupeAtMost says 4)
k = 2, a = [1 1 1 2 2 3 3 3 3 4]
  read write  a[read] a[write-k]  keep?  a[:write] after the step
     0     0       1         -  true   [1]
     1     1       1         -  true   [1 1]
     2     2       1         1  false  [1 1]
     3     2       2         1  true   [1 1 2]
     4     3       2         1  true   [1 1 2 2]
     5     4       3         2  true   [1 1 2 2 3]
     6     5       3         2  true   [1 1 2 2 3 3]
     7     6       3         3  false  [1 1 2 2 3 3]
     8     6       3         3  false  [1 1 2 2 3 3]
     9     6       4         3  true   [1 1 2 2 3 3 4]
  invariant held at 10 of 10 steps; final length 7 (dedupeAtMost says 7)

== (b) counts on sorted random data (values IntN(n/4)) ==
          n  k     reads    writes      kept   allocs  matches oracle
      10000  1     10000      2458      2458        0  true
      10000  2     10000      4736      4736        0  true
      10000  3     10000      6642      6642        0  true
    1000000  1   1000000    245311    245311        0  true
    1000000  2   1000000    472249    472249        0  true
    1000000  3   1000000    662625    662625        0  true

== (c) the unsorted trap ==
  unsorted [1 2 1 2 1], k = 1: length 5, slice now [1 2 1 2 1] (no two neighbours are equal)
  sorted first:                length 2, slice now [1 2]

== (d) 20000 random sorted arrays (length <= 20, values IntN(6), k in 1..3) ==
  versus the counting oracle: 0 mismatches; k = 1 versus slices.Compact: 0 mismatches in 6534 cases

== (e) what is left beyond write ==
  by hand:        len 4, answer [1 2 3 4]
                  backing array [1 2 3 4 2 3 3 3 3 4]  <- the tail is stale
  slices.Compact: len 4, answer [1 2 3 4]
                  backing array [1 2 3 4 0 0 0 0 0 0]  <- the tail is zeroed
  (read in the source, not run: Compact ends with clear(s[k:]) for that reason; with
  pointer elements a stale tail would also keep the pointed-to values alive.)
```

**Rust:**

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::collections::HashMap;
use std::sync::atomic::{AtomicUsize, Ordering::Relaxed};

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) feeds this program the same numbers that Go's
// rand.New(rand.NewPCG(1, 2)) feeds the Go program.
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

// A tiny counting allocator: stand-in for testing.AllocsPerRun. It counts
// allocation events (alloc, alloc_zeroed and realloc) for the whole process.
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
    unsafe fn realloc(&self, p: *mut u8, l: Layout, new: usize) -> *mut u8 {
        ALLOCS.fetch_add(1, Relaxed);
        unsafe { System.realloc(p, l, new) }
    }
}
#[global_allocator]
static GLOBAL: Counting = Counting;

/// allocation events while running `f`.
fn allocs_of<R>(f: impl FnOnce() -> R) -> (usize, R) {
    let a0 = ALLOCS.load(Relaxed);
    let r = f();
    (ALLOCS.load(Relaxed) - a0, r)
}

// dedupe_at_most keeps at most k copies of each value, IN PLACE, in a sorted
// slice, and returns (new length, reads, writes).
//
// Invariant: a[..write] is the answer for a[..read]. The test looks k places
// back in the OUTPUT (a[write-k]), not the input: if the last k kept values
// all equal a[read], then keeping it would be copy number k+1.
fn dedupe_at_most(a: &mut [i64], k: usize) -> (usize, usize, usize) {
    if k < 1 {
        // "at most 0 copies" keeps nothing
        return (0, 0, 0);
    }
    let (mut write, mut reads, mut writes) = (0, 0, 0);
    for read in 0..a.len() {
        reads += 1;
        // `write < k` guards a[write - k]: with a usize, write - k would
        // underflow there (Go would just compute a negative index it never uses).
        if write < k || a[read] != a[write - k] {
            a[write] = a[read];
            write += 1;
            writes += 1;
        }
    }
    (write, reads, writes)
}

// keep_first_k is the oracle: no pointers, a map of counts. It builds a new Vec.
fn keep_first_k(a: &[i64], k: usize) -> Vec<i64> {
    let mut seen: HashMap<i64, usize> = HashMap::new();
    let mut out = Vec::new();
    for &v in a {
        let c = seen.entry(v).or_insert(0);
        if *c < k {
            out.push(v);
        }
        *c += 1;
    }
    out
}

// Go prints a slice as [1 3 4]; Rust's {:?} prints [1, 3, 4]. Format by hand.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

// trace is the same loop with a print per step. After every step it checks
// the invariant: a[..write] must equal the from-scratch answer for a0[..=read].
fn trace(a0: &[i64], k: usize) {
    let mut a = a0.to_vec();
    println!("k = {k}, a = {}", go_slice(a0));
    println!("  read write  a[read] a[write-k]  keep?  a[:write] after the step");
    let (mut write, mut held) = (0, 0);
    for read in 0..a.len() {
        let v = a[read];
        let (back, keep) = if write >= k {
            (a[write - k].to_string(), v != a[write - k])
        } else {
            ("-".to_string(), true)
        };
        let w = write;
        if keep {
            a[write] = v;
            write += 1;
        }
        if a[..write] == keep_first_k(&a0[..=read], k)[..] {
            held += 1;
        }
        println!(
            "  {read:>4} {w:>5}  {v:>6} {back:>9}  {keep:<5}  {}",
            go_slice(&a[..write])
        );
    }
    let (n, _, _) = dedupe_at_most(&mut a0.to_vec(), k);
    println!(
        "  invariant held at {held} of {} steps; final length {write} (dedupe_at_most says {n})",
        a0.len()
    );
}

fn main() {
    println!("== (a) the trace ==");
    let a0 = [1, 1, 1, 2, 2, 3, 3, 3, 3, 4];
    trace(&a0, 1);
    trace(&a0, 2);

    println!();
    println!("== (b) counts on sorted random data (values IntN(n/4)) ==");
    let mut rng = Pcg::new(1, 2);
    println!(
        "  {:>9} {:>2} {:>9} {:>9} {:>9} {:>8}  matches oracle",
        "n", "k", "reads", "writes", "kept", "allocs"
    );
    for n in [10_000usize, 1_000_000] {
        let mut src: Vec<i64> = (0..n).map(|_| rng.int_n(n as u64 / 4) as i64).collect();
        src.sort();
        let mut buf = vec![0i64; n];
        for k in 1..=3 {
            let (allocs, (kept, reads, writes)) = allocs_of(|| {
                buf.copy_from_slice(&src); // restore the input; no allocation
                dedupe_at_most(&mut buf, k)
            });
            let ok = buf[..kept] == keep_first_k(&src, k)[..];
            println!("  {n:>9} {k:>2} {reads:>9} {writes:>9} {kept:>9} {allocs:>8}  {ok}");
        }
    }

    println!();
    println!("== (c) the unsorted trap ==");
    let mut u = [1, 2, 1, 2, 1];
    let (n, _, _) = dedupe_at_most(&mut u, 1);
    println!(
        "  unsorted [1 2 1 2 1], k = 1: length {n}, slice now {} (no two neighbours are equal)",
        go_slice(&u[..n])
    );
    u.sort();
    let (n, _, _) = dedupe_at_most(&mut u, 1);
    println!(
        "  sorted first:                length {n}, slice now {}",
        go_slice(&u[..n])
    );

    println!();
    println!("== (d) 20000 random sorted arrays (length <= 20, values IntN(6), k in 1..3) ==");
    let (mut bad, mut ones, mut bad_dedup) = (0, 0, 0);
    for _ in 0..20000 {
        let len = rng.int_n(21) as usize;
        let mut a: Vec<i64> = (0..len).map(|_| rng.int_n(6) as i64).collect();
        a.sort();
        let k = 1 + rng.int_n(3) as usize;
        let want = keep_first_k(&a, k);
        let mut std_dedup = a.clone();
        if k == 1 {
            ones += 1;
            std_dedup.dedup(); // the standard library's version
        }
        let (n, _, _) = dedupe_at_most(&mut a, k);
        if a[..n] != want[..] {
            bad += 1;
        }
        if k == 1 && a[..n] != std_dedup[..] {
            bad_dedup += 1;
        }
    }
    println!(
        "  versus the counting oracle: {bad} mismatches; k = 1 versus Vec::dedup: {bad_dedup} mismatches in {ones} cases"
    );

    println!();
    println!("== (e) what is left beyond write ==");
    let mut a = a0;
    let (n, _, _) = dedupe_at_most(&mut a, 1);
    println!("  by hand:        len {n}, answer {}", go_slice(&a[..n]));
    println!(
        "                  backing array {}  <- the tail is stale",
        go_slice(&a)
    );
    a[n..].fill(0); // what slices.Compact does in Go: clear the tail yourself
    println!(
        "  + fill(0):      backing array {}  <- zeroed by hand",
        go_slice(&a)
    );
    let mut v = a0.to_vec();
    v.dedup();
    println!(
        "  Vec::dedup:     len {}, answer {}, capacity {}  <- the removed elements are dropped; the",
        v.len(),
        go_slice(&v),
        v.capacity()
    );
    println!("                  slots past len exist but safe code cannot read them");
    println!(
        "  (read in the source, not run: dedup_by is this same read/write loop, with the comment"
    );
    println!(
        "  `INVARIANT: vec.len() > read > write > write-1 >= 0`; it drops the removed elements"
    );
    println!("  and ends with set_len(write).)");
}
```

**Rust output:**

```
== (a) the trace ==
k = 1, a = [1 1 1 2 2 3 3 3 3 4]
  read write  a[read] a[write-k]  keep?  a[:write] after the step
     0     0       1         -  true   [1]
     1     1       1         1  false  [1]
     2     1       1         1  false  [1]
     3     1       2         1  true   [1 2]
     4     2       2         2  false  [1 2]
     5     2       3         2  true   [1 2 3]
     6     3       3         3  false  [1 2 3]
     7     3       3         3  false  [1 2 3]
     8     3       3         3  false  [1 2 3]
     9     3       4         3  true   [1 2 3 4]
  invariant held at 10 of 10 steps; final length 4 (dedupe_at_most says 4)
k = 2, a = [1 1 1 2 2 3 3 3 3 4]
  read write  a[read] a[write-k]  keep?  a[:write] after the step
     0     0       1         -  true   [1]
     1     1       1         -  true   [1 1]
     2     2       1         1  false  [1 1]
     3     2       2         1  true   [1 1 2]
     4     3       2         1  true   [1 1 2 2]
     5     4       3         2  true   [1 1 2 2 3]
     6     5       3         2  true   [1 1 2 2 3 3]
     7     6       3         3  false  [1 1 2 2 3 3]
     8     6       3         3  false  [1 1 2 2 3 3]
     9     6       4         3  true   [1 1 2 2 3 3 4]
  invariant held at 10 of 10 steps; final length 7 (dedupe_at_most says 7)

== (b) counts on sorted random data (values IntN(n/4)) ==
          n  k     reads    writes      kept   allocs  matches oracle
      10000  1     10000      2458      2458        0  true
      10000  2     10000      4736      4736        0  true
      10000  3     10000      6642      6642        0  true
    1000000  1   1000000    245311    245311        0  true
    1000000  2   1000000    472249    472249        0  true
    1000000  3   1000000    662625    662625        0  true

== (c) the unsorted trap ==
  unsorted [1 2 1 2 1], k = 1: length 5, slice now [1 2 1 2 1] (no two neighbours are equal)
  sorted first:                length 2, slice now [1 2]

== (d) 20000 random sorted arrays (length <= 20, values IntN(6), k in 1..3) ==
  versus the counting oracle: 0 mismatches; k = 1 versus Vec::dedup: 0 mismatches in 6534 cases

== (e) what is left beyond write ==
  by hand:        len 4, answer [1 2 3 4]
                  backing array [1 2 3 4 2 3 3 3 3 4]  <- the tail is stale
  + fill(0):      backing array [1 2 3 4 0 0 0 0 0 0]  <- zeroed by hand
  Vec::dedup:     len 4, answer [1 2 3 4], capacity 10  <- the removed elements are dropped; the
                  slots past len exist but safe code cannot read them
  (read in the source, not run: dedup_by is this same read/write loop, with the comment
  `INVARIANT: vec.len() > read > write > write-1 >= 0`; it drops the removed elements
  and ends with set_len(write).)
```

> **Go vs Rust:** Every count is identical to Go (reads, writes, kept lengths, 0 allocations, 6,534 `k = 1` cases, 0 mismatches). `Vec::dedup` is the same read/write loop for k = 1 but it shortens `len` and drops the removed elements, so there is no stale tail to print (the capacity stays 10, the slots are just unreadable); the hand-written `&mut [i64]` version leaves the tail stale exactly as Go does, and `fill(0)` is what `slices.Compact` does for you. `a[write - k]` is the Rust-visible edge: `write < k ||` is what stops the `usize` subtraction from underflowing, and the allocation count comes from a 15-line counting global allocator instead of `testing.AllocsPerRun`.

**Complexity:** Θ(n) time, Θ(1) extra space. Exactly n reads (10,000 and 1,000,000 above), at most n writes, 0 allocations measured with `testing.AllocsPerRun`, and the comparison is with `a[write-k]` in the output. It needs equal values to be adjacent: `[1 2 1 2 1]` is returned unchanged until it is sorted (a Θ(n log n) step). The hand-written loop leaves a stale tail beyond `write`; `slices.Compact` zeroes it.

---

## 3. A fixed window: add one, drop one

`🟢 easy` · *Fixed sliding window*

The maximum sum of k consecutive elements. Recomputing every window costs k additions each, Θ(nk) in all; sliding the window one step right adds the element that enters and subtracts the one that leaves, 2 updates per window whatever k is, Θ(n) in all. It works because `+` has an inverse. (Lesson 09 example 10 needed a monotonic deque for the window *maximum* precisely because max has none.) With floating point the inverse is only approximate, and the last section shows the damage.

**Steps:**

1. Trace the slide on `[2 1 5 1 3 2]` with k = 3: which element enters, which leaves, and what the running sum and the best become.
2. Count the additions for recomputing against sliding at n = 100,000 and k from 10 to 10,000.
3. Check the slide against brute force on random inputs including `k == 0`, `k > n` and `k == n`, then feed it floats where an exact answer is lost.

```go
package main

import (
	"fmt"
	"math/rand/v2"
)

// Every function counts the additions and subtractions it applies to
// elements ("adds"). A window needs 1 <= k <= len(a); otherwise there is no
// window and ok is false (no panic).

// recompute sums every window from scratch: (n-k+1)*k additions.
func recompute(a []int, k int) (best, at, adds int, ok bool) {
	if k < 1 || k > len(a) {
		return 0, -1, 0, false
	}
	for i := 0; i+k <= len(a); i++ {
		s := 0
		for _, v := range a[i : i+k] {
			s += v
			adds++
		}
		if !ok || s > best {
			best, at, ok = s, i, true
		}
	}
	return best, at, adds, ok
}

// slide fills the first window (k additions), then moves it one step per
// iteration: add the element that enters, subtract the one that leaves.
func slide(a []int, k int) (best, at, adds int, ok bool) {
	if k < 1 || k > len(a) {
		return 0, -1, 0, false
	}
	s := 0
	for _, v := range a[:k] {
		s += v
		adds++
	}
	best, at = s, 0
	for r := k; r < len(a); r++ { // r is the entering index, r-k the leaving one
		s += a[r]
		s -= a[r-k]
		adds += 2
		if s > best {
			best, at = s, r-k+1
		}
	}
	return best, at, adds, true
}

func main() {
	fmt.Println("== (a) the trace: [2 1 5 1 3 2], k = 3 ==")
	a := []int{2, 1, 5, 1, 3, 2}
	k := 3
	s := 0
	for _, v := range a[:k] {
		s += v
	}
	best, at := s, 0
	fmt.Println("  window   add drop  sum  best (at)")
	fmt.Printf("  [%d,%d)    -    -  %3d  %4d (%d)\n", 0, k, s, best, at)
	for r := k; r < len(a); r++ {
		s += a[r] - a[r-k]
		if s > best {
			best, at = s, r-k+1
		}
		fmt.Printf("  [%d,%d)  %4d %4d  %3d  %4d (%d)\n", r-k+1, r+1, a[r], a[r-k], s, best, at)
	}

	fmt.Println()
	fmt.Println("== (b) additions at n = 100000, values IntN(1000)-500 ==")
	rng := rand.New(rand.NewPCG(1, 2))
	const n = 100_000
	big := make([]int, n)
	for i := range big {
		big[i] = rng.IntN(1000) - 500
	}
	fmt.Printf("  %6s %14s %12s %9s  %s\n", "k", "recompute", "slide", "ratio", "formulas hold, same answer")
	for _, k := range []int{10, 100, 1000, 10000} {
		b1, at1, adds1, _ := recompute(big, k)
		b2, at2, adds2, _ := slide(big, k)
		fmt.Printf("  %6d %14d %12d %8.1fx  %v\n", k, adds1, adds2, float64(adds1)/float64(adds2),
			adds1 == (n-k+1)*k && adds2 == k+2*(n-k) && b1 == b2 && at1 == at2)
	}

	fmt.Println()
	fmt.Println("== (c) 5000 random cases against brute force ==")
	var zero, tooBig, whole, normal, bad int
	for range 5000 {
		xs := make([]int, rng.IntN(16))
		for i := range xs {
			xs[i] = rng.IntN(11) - 5
		}
		k := rng.IntN(17)
		switch {
		case k == 0:
			zero++
		case k > len(xs):
			tooBig++
		case k == len(xs):
			whole++
		default:
			normal++
		}
		b1, at1, _, ok1 := recompute(xs, k)
		b2, at2, _, ok2 := slide(xs, k)
		if b1 != b2 || at1 != at2 || ok1 != ok2 {
			bad++
		}
	}
	fmt.Printf("  k == 0: %d, k > n: %d, k == n: %d, 0 < k < n: %d cases; %d mismatches\n", zero, tooBig, whole, normal, bad)

	fmt.Println()
	fmt.Println("== (d) floats: add-then-subtract is not exact ==")
	f := []float64{1e16, 1, 1, 1, 1, 1}
	g := []int64{1e16, 1, 1, 1, 1, 1} // the same data as integers
	const w = 3
	// The sliding sum is  s += f[r] - f[r-w] : Go evaluates the right-hand side
	// first (the difference), then adds it to s.
	fs := f[0] + f[1] + f[2]
	is := g[0] + g[1] + g[2]
	fmt.Printf("  %-8s %17s %17s %17s %9s %11s\n", "window", "float slide", "float recompute", "exact (int64)", "int slide", "slide error")
	first, intExact := "", true
	for i := 0; i+w <= len(f); i++ {
		if i > 0 {
			fs += f[i+w-1] - f[i-1]
			is += g[i+w-1] - g[i-1]
		}
		re := f[i] + f[i+1] + f[i+2]
		ex := g[i] + g[i+1] + g[i+2]
		if first == "" && fs != re {
			first = fmt.Sprintf("[%d,%d)", i, i+w)
		}
		intExact = intExact && is == ex
		fmt.Printf("  [%d,%d)   %17.0f %17.0f %17d %9d %11.0f\n", i, i+w, fs, re, ex, is, fs-float64(ex))
	}
	fmt.Printf("  first window where the float slide and the float recompute differ: %s\n", first)
	fmt.Printf("  int slide equals the exact sum in every window: %v\n", intExact)
	fmt.Printf("  why: 1e16 + 1 == 1e16 is %v and 1 - 1e16 == -1e16 is %v in float64\n", f[0]+f[1] == f[0], f[1]-f[0] == -f[0])
}
```

**Output:**

```
== (a) the trace: [2 1 5 1 3 2], k = 3 ==
  window   add drop  sum  best (at)
  [0,3)    -    -    8     8 (0)
  [1,4)     1    2    7     8 (0)
  [2,5)     3    1    9     9 (2)
  [3,6)     2    5    6     9 (2)

== (b) additions at n = 100000, values IntN(1000)-500 ==
       k      recompute        slide     ratio  formulas hold, same answer
      10         999910       199990      5.0x  true
     100        9990100       199900     50.0x  true
    1000       99001000       199000    497.5x  true
   10000      900010000       190000   4736.9x  true

== (c) 5000 random cases against brute force ==
  k == 0: 315, k > n: 2503, k == n: 281, 0 < k < n: 1901 cases; 0 mismatches

== (d) floats: add-then-subtract is not exact ==
  window         float slide   float recompute     exact (int64) int slide slide error
  [0,3)   10000000000000000 10000000000000000 10000000000000002 10000000000000002          -2
  [1,4)                   0                 3                 3         3          -3
  [2,5)                   0                 3                 3         3          -3
  [3,6)                   0                 3                 3         3          -3
  first window where the float slide and the float recompute differ: [1,4)
  int slide equals the exact sum in every window: true
  why: 1e16 + 1 == 1e16 is true and 1 - 1e16 == -1e16 is true in float64
```

**Rust:**

```rust
use std::cell::Cell;

// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) feeds this program the same numbers that Go's
// rand.New(rand.NewPCG(1, 2)) feeds the Go program.
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

// Every function counts the additions and subtractions it applies to
// elements ("adds"). A window needs 1 <= k <= len; otherwise there is no
// window and the result is None (no panic).

// recompute sums every window from scratch: (n-k+1)*k additions. It uses the
// idiomatic `windows(k)` + `sum()`, with `inspect` counting each element added.
fn recompute(a: &[i64], k: usize) -> (Option<(i64, usize)>, u64) {
    if k < 1 {
        // windows(0) panics ("window size must be non-zero"), so guard it.
        // windows(k) with k > len just yields no windows.
        return (None, 0);
    }
    let adds = Cell::new(0u64);
    let mut best: Option<(i64, usize)> = None;
    for (i, w) in a.windows(k).enumerate() {
        let s: i64 = w.iter().inspect(|_| adds.set(adds.get() + 1)).sum();
        if best.is_none_or(|(b, _)| s > b) {
            best = Some((s, i));
        }
    }
    (best, adds.get())
}

// slide fills the first window (k additions), then moves it one step per
// iteration: add the element that enters, subtract the one that leaves.
fn slide(a: &[i64], k: usize) -> (Option<(i64, usize)>, u64) {
    if k < 1 || k > a.len() {
        return (None, 0);
    }
    let (mut s, mut adds) = (0, 0);
    for &v in &a[..k] {
        s += v;
        adds += 1;
    }
    let (mut best, mut at) = (s, 0);
    for r in k..a.len() {
        // r is the entering index, r-k the leaving one
        s += a[r];
        s -= a[r - k];
        adds += 2;
        if s > best {
            best = s;
            at = r - k + 1;
        }
    }
    (Some((best, at)), adds)
}

fn main() {
    println!("== (a) the trace: [2 1 5 1 3 2], k = 3 ==");
    let a = [2i64, 1, 5, 1, 3, 2];
    let k = 3;
    let mut s: i64 = a[..k].iter().sum();
    let (mut best, mut at) = (s, 0);
    println!("  window   add drop  sum  best (at)");
    println!("  [{},{})    -    -  {s:>3}  {best:>4} ({at})", 0, k);
    for r in k..a.len() {
        s += a[r] - a[r - k];
        if s > best {
            best = s;
            at = r - k + 1;
        }
        println!(
            "  [{},{})  {:>4} {:>4}  {s:>3}  {best:>4} ({at})",
            r - k + 1,
            r + 1,
            a[r],
            a[r - k]
        );
    }

    println!();
    println!("== (b) additions at n = 100000, values IntN(1000)-500 ==");
    let mut rng = Pcg::new(1, 2);
    const N: usize = 100_000;
    let big: Vec<i64> = (0..N).map(|_| rng.int_n(1000) as i64 - 500).collect();
    println!(
        "  {:>6} {:>14} {:>12} {:>9}  formulas hold, same answer",
        "k", "recompute", "slide", "ratio"
    );
    for k in [10usize, 100, 1000, 10000] {
        let (r1, adds1) = recompute(&big, k);
        let (r2, adds2) = slide(&big, k);
        let holds =
            adds1 == ((N - k + 1) * k) as u64 && adds2 == (k + 2 * (N - k)) as u64 && r1 == r2;
        println!(
            "  {k:>6} {adds1:>14} {adds2:>12} {:>8.1}x  {holds}",
            adds1 as f64 / adds2 as f64
        );
    }

    println!();
    println!("== (c) 5000 random cases against brute force ==");
    let (mut zero, mut too_big, mut whole, mut normal, mut bad) = (0, 0, 0, 0, 0);
    for _ in 0..5000 {
        let len = rng.int_n(16) as usize;
        let xs: Vec<i64> = (0..len).map(|_| rng.int_n(11) as i64 - 5).collect();
        let k = rng.int_n(17) as usize;
        match k {
            0 => zero += 1,
            _ if k > xs.len() => too_big += 1,
            _ if k == xs.len() => whole += 1,
            _ => normal += 1,
        }
        let (r1, _) = recompute(&xs, k);
        let (r2, _) = slide(&xs, k);
        // the one-liner says the same thing about the best sum
        let one_liner = if k == 0 {
            None
        } else {
            xs.windows(k).map(|w| w.iter().sum::<i64>()).max()
        };
        if r1 != r2 || r1.map(|(b, _)| b) != one_liner {
            bad += 1;
        }
    }
    println!(
        "  k == 0: {zero}, k > n: {too_big}, k == n: {whole}, 0 < k < n: {normal} cases; {bad} mismatches"
    );
    println!(
        "  (extra, Rust only: [1, 2, 3].windows(5).count() = {})",
        [1, 2, 3].windows(5).count()
    );

    println!();
    println!("== (d) floats: add-then-subtract is not exact ==");
    let f = [1e16f64, 1.0, 1.0, 1.0, 1.0, 1.0];
    let g = [10_000_000_000_000_000i64, 1, 1, 1, 1, 1]; // the same data as integers
    const W: usize = 3;
    // The sliding sum is  s += f[r] - f[r-w] : the right-hand side is evaluated
    // first (the difference), then added to s, as in Go.
    let mut fs = f[0] + f[1] + f[2];
    let mut si = g[0] + g[1] + g[2];
    println!(
        "  {:<8} {:>17} {:>17} {:>17} {:>9} {:>11}",
        "window", "float slide", "float recompute", "exact (int64)", "int slide", "slide error"
    );
    let mut first = String::new();
    let mut int_exact = true;
    for i in 0..=f.len() - W {
        if i > 0 {
            fs += f[i + W - 1] - f[i - 1];
            si += g[i + W - 1] - g[i - 1];
        }
        let re = f[i] + f[i + 1] + f[i + 2];
        let ex = g[i] + g[i + 1] + g[i + 2];
        if first.is_empty() && fs != re {
            first = format!("[{},{})", i, i + W);
        }
        int_exact = int_exact && si == ex;
        println!(
            "  [{},{})   {fs:>17.0} {re:>17.0} {ex:>17} {si:>9} {:>11.0}",
            i,
            i + W,
            fs - ex as f64
        );
    }
    println!("  first window where the float slide and the float recompute differ: {first}");
    println!("  int slide equals the exact sum in every window: {int_exact}");
    println!(
        "  why: 1e16 + 1 == 1e16 is {} and 1 - 1e16 == -1e16 is {} in float64",
        f[0] + f[1] == f[0],
        f[1] - f[0] == -f[0]
    );
}
```

**Rust output:**

```
== (a) the trace: [2 1 5 1 3 2], k = 3 ==
  window   add drop  sum  best (at)
  [0,3)    -    -    8     8 (0)
  [1,4)     1    2    7     8 (0)
  [2,5)     3    1    9     9 (2)
  [3,6)     2    5    6     9 (2)

== (b) additions at n = 100000, values IntN(1000)-500 ==
       k      recompute        slide     ratio  formulas hold, same answer
      10         999910       199990      5.0x  true
     100        9990100       199900     50.0x  true
    1000       99001000       199000    497.5x  true
   10000      900010000       190000   4736.9x  true

== (c) 5000 random cases against brute force ==
  k == 0: 315, k > n: 2503, k == n: 281, 0 < k < n: 1901 cases; 0 mismatches
  (extra, Rust only: [1, 2, 3].windows(5).count() = 0)

== (d) floats: add-then-subtract is not exact ==
  window         float slide   float recompute     exact (int64) int slide slide error
  [0,3)   10000000000000000 10000000000000000 10000000000000002 10000000000000002          -2
  [1,4)                   0                 3                 3         3          -3
  [2,5)                   0                 3                 3         3          -3
  [3,6)                   0                 3                 3         3          -3
  first window where the float slide and the float recompute differ: [1,4)
  int slide equals the exact sum in every window: true
  why: 1e16 + 1 == 1e16 is true and 1 - 1e16 == -1e16 is true in float64
```

> **Go vs Rust:** Identical counts and numbers to Go, including the float table (both are IEEE-754 binary64: the slide is stuck at 0 once `1e16` leaves). The idiomatic-looking `a.windows(k).map(|w| w.iter().sum::<i64>())` is the **recompute** column: it does (n−k+1)·k additions, counted here with `inspect`, so it is not the slide. `windows(0)` panics and `windows(k)` with `k > len` yields nothing (the extra line prints 0 windows), hence the explicit `k < 1` guard; the result is an `Option<(i64, usize)>` instead of Go's `ok` flag. Go has no windows helper (`slices.Chunk` does not overlap), so it indexes.

**Complexity:** Θ(n) time, Θ(1) space. The slide does k + 2(n−k) additions: 199,990 at k = 10 and 190,000 at k = 10,000 (it falls slightly as k grows, because only the first window costs k), while recomputing does (n−k+1)·k, from 999,910 to 900,010,000. It needs an invertible aggregate, and in floating point add-then-subtract is not invertible: once `1e16` left the window the float slide stayed at 0 where the true sums were 3.

---

## 4. Prefix sums: a range sum in Θ(1)

`🟢 easy` · *Prefix sums*

`P[0] = 0` and `P[i+1] = P[i] + a[i]`, so the sum of the half-open range `a[l:r]` is `P[r] − P[l]`. The zero in slot 0 deletes the `l == 0` special case and the half-open range makes the empty range free. Building P costs Θ(n) once; after that every query is two reads, Θ(1) — which pays off only for data that does not change, and the program counts both halves of that sentence.

**Steps:**

1. Build P for a small array, answer four queries with the formula spelled out, and check each against a direct sum.
2. Run every one of the 45 ranges of an 8-element array through the sentinel-free formula (and through a second classic mistake) and classify the results.
3. Count element reads for q queries at n = 100,000 to find where the prefix array starts to win, then add updates and watch it lose.

```go
package main

import (
	"fmt"
	"math/rand/v2"
)

// prefix returns P with len(a)+1 entries: P[0] = 0, P[i+1] = P[i] + a[i].
// So P[i] is the sum of the first i elements, and a[l:r] sums to P[r] - P[l].
func prefix(a []int) []int {
	p := make([]int, len(a)+1)
	for i, v := range a {
		p[i+1] = p[i] + v
	}
	return p
}

// direct sums a[l:r] by looping: the oracle, and the cost the prefix array avoids.
func direct(a []int, l, r int) int {
	s := 0
	for _, v := range a[l:r] {
		s += v
	}
	return s
}

// randRange returns a random half-open range 0 <= l <= r <= n.
func randRange(rng *rand.Rand, n int) (l, r int) {
	l, r = rng.IntN(n+1), rng.IntN(n+1)
	if l > r {
		l, r = r, l
	}
	return l, r
}

func main() {
	fmt.Println("== (a) the formula ==")
	a := []int{3, 1, 4, 1, 5, 9, 2, 6}
	p := prefix(a)
	fmt.Println("  a =", a)
	fmt.Println("  P =", p)
	for _, q := range [][2]int{{0, 8}, {2, 5}, {3, 3}, {7, 8}} {
		l, r := q[0], q[1]
		fmt.Printf("  a[%d:%d] = P[%d] - P[%d] = %d - %d = %d   (direct sum: %d)\n",
			l, r, r, l, p[r], p[l], p[r]-p[l], direct(a, l, r))
	}

	fmt.Println()
	fmt.Println("== (b) the off-by-one, on every range 0 <= l <= r <= 8 ==")
	// inc[i] = a[0]+...+a[i] has only n entries: no sentinel slot.
	inc := make([]int, len(a))
	for i, v := range a {
		inc[i] = v
		if i > 0 {
			inc[i] += inc[i-1]
		}
	}
	n := len(a)
	variants := []struct {
		name string
		eval func(l, r int) (sum int, inRange bool) // inRange: were all the indexes valid?
	}{
		{"inc[r-1] - inc[l-1], no sentinel", func(l, r int) (int, bool) {
			if l-1 < 0 || r-1 < 0 { // the check comes BEFORE the read: Go would panic on inc[-1]
				return 0, false
			}
			return inc[r-1] - inc[l-1], true
		}},
		{"inc[r] - inc[l], same array", func(l, r int) (int, bool) {
			if r >= n {
				return 0, false
			}
			return inc[r] - inc[l], true
		}},
		{"P[r] - P[l], sentinel P[0] = 0", func(l, r int) (int, bool) { return p[r] - p[l], true }},
	}
	fmt.Printf("  %-34s %7s %6s %6s %13s %9s\n", "formula", "ranges", "right", "wrong", "out of range", "non-empty")
	for _, v := range variants {
		var total, right, wrong, out, rightNonEmpty int
		for l := 0; l <= n; l++ {
			for r := l; r <= n; r++ {
				total++
				switch sum, ok := v.eval(l, r); {
				case !ok:
					out++
				case sum == direct(a, l, r):
					right++
					if l < r {
						rightNonEmpty++
					}
				default:
					wrong++
				}
			}
		}
		fmt.Printf("  %-34s %7d %6d %6d %13d %9d\n", v.name, total, right, wrong, out, rightNonEmpty)
	}
	fmt.Println("  (last column: how many of the right answers were for non-empty ranges; there are 36 of those)")

	fmt.Println()
	fmt.Println("== (c) element reads: re-summing vs a prefix array, n = 100000 ==")
	rng := rand.New(rand.NewPCG(1, 2))
	const N = 100_000
	big := make([]int, N)
	for i := range big {
		big[i] = rng.IntN(1000) - 500
	}
	pb := prefix(big)
	type rg struct{ l, r int }
	ranges := make([]rg, N)
	for i := range ranges {
		ranges[i].l, ranges[i].r = randRange(rng, N)
	}
	// First 1000 queries for real: the answers agree, and the elements we
	// actually add are the sum of (r-l) that the table below counts arithmetically.
	agree, looped, formula := true, 0, 0
	for _, q := range ranges[:1000] {
		s := 0
		for _, v := range big[q.l:q.r] {
			s += v
			looped++
		}
		formula += q.r - q.l
		agree = agree && s == pb[q.r]-pb[q.l]
	}
	fmt.Printf("  first 1000 queries: answers agree %v, elements actually added %d, sum of lengths %d\n", agree, looped, formula)
	fmt.Printf("  %8s %16s %14s  %s\n", "q", "re-sum reads", "prefix reads", "cheaper")
	checkpoints := map[int]bool{1: true, 2: true, 3: true, 5: true, 10: true, 100: true, 1000: true, 100000: true}
	sum, firstWin, lastLoss := 0, 0, 0
	for q := 1; q <= N; q++ {
		sum += ranges[q-1].r - ranges[q-1].l
		prefixReads := N + 2*q // N to build, then two reads per query
		if prefixReads < sum && firstWin == 0 {
			firstWin = q
		}
		if sum <= prefixReads {
			lastLoss = q
		}
		if checkpoints[q] {
			who := "prefix"
			if sum <= prefixReads {
				who = "re-sum"
			}
			fmt.Printf("  %8d %16d %14d  %s\n", q, sum, prefixReads, who)
		}
	}
	fmt.Printf("  the prefix array first wins at q = %d; re-summing was last ahead at q = %d\n", firstWin, lastLoss)

	fmt.Println()
	fmt.Println("== (d) 1000 interleaved update + query pairs at n = 100000 ==")
	// Each pair: a[i] += delta, then ask a range sum. Three ways to answer it.
	work := append([]int(nil), big...) // plain array, updated in place
	pFull := prefix(big)               // rebuilt from scratch after every update
	pPart := prefix(big)               // only P[i+1:] repaired after every update
	var readsFull, readsPart, readsSum, wrong int
	for range 1000 {
		i, delta := rng.IntN(N), rng.IntN(21)-10
		l, r := randRange(rng, N)
		work[i] += delta

		for j := range N { // rebuild everything: N reads
			pFull[j+1] = pFull[j] + work[j]
			readsFull++
		}
		for j := i; j < N; j++ { // P[:i+1] is still valid: N-i reads
			pPart[j+1] = pPart[j] + work[j]
			readsPart++
		}
		s := 0
		for j := l; j < r; j++ { // no prefix array at all: r-l reads
			s += work[j]
			readsSum++
		}
		readsFull += 2
		readsPart += 2
		if pFull[r]-pFull[l] != s || pPart[r]-pPart[l] != s {
			wrong++
		}
	}
	fmt.Printf("  %-34s %14s %9s\n", "strategy", "element reads", "vs re-sum")
	fmt.Printf("  %-34s %14d %8.1fx\n", "rebuild all of P after each update", readsFull, float64(readsFull)/float64(readsSum))
	fmt.Printf("  %-34s %14d %8.1fx\n", "repair only P[i+1:] after each one", readsPart, float64(readsPart)/float64(readsSum))
	fmt.Printf("  %-34s %14d %8.1fx\n", "no prefix array, re-sum the range", readsSum, 1.0)
	fmt.Printf("  wrong answers: %d. A prefix array is for data that does not change;\n", wrong)
	fmt.Println("  for updates plus range sums the structure is a Fenwick tree (not covered here).")
}
```

**Output:**

```
== (a) the formula ==
  a = [3 1 4 1 5 9 2 6]
  P = [0 3 4 8 9 14 23 25 31]
  a[0:8] = P[8] - P[0] = 31 - 0 = 31   (direct sum: 31)
  a[2:5] = P[5] - P[2] = 14 - 4 = 10   (direct sum: 10)
  a[3:3] = P[3] - P[3] = 8 - 8 = 0   (direct sum: 0)
  a[7:8] = P[8] - P[7] = 31 - 25 = 6   (direct sum: 6)

== (b) the off-by-one, on every range 0 <= l <= r <= 8 ==
  formula                             ranges  right  wrong  out of range non-empty
  inc[r-1] - inc[l-1], no sentinel        45     36      0             9        28
  inc[r] - inc[l], same array             45      9     27             9         1
  P[r] - P[l], sentinel P[0] = 0          45     45      0             0        36
  (last column: how many of the right answers were for non-empty ranges; there are 36 of those)

== (c) element reads: re-summing vs a prefix array, n = 100000 ==
  first 1000 queries: answers agree true, elements actually added 33220513, sum of lengths 33220513
         q     re-sum reads   prefix reads  cheaper
         1            45695         100002  re-sum
         2            61590         100004  re-sum
         3           148620         100006  prefix
         5           241650         100010  prefix
        10           464009         100020  prefix
       100          3367651         100200  prefix
      1000         33220513         102000  prefix
    100000       3332758140         300000  prefix
  the prefix array first wins at q = 3; re-summing was last ahead at q = 2

== (d) 1000 interleaved update + query pairs at n = 100000 ==
  strategy                            element reads vs re-sum
  rebuild all of P after each update      100002000      2.9x
  repair only P[i+1:] after each one       50732343      1.5x
  no prefix array, re-sum the range        34099009      1.0x
  wrong answers: 0. A prefix array is for data that does not change;
  for updates plus range sums the structure is a Fenwick tree (not covered here).
```

**Rust:**

```rust
// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) feeds this program the same numbers that Go's
// rand.New(rand.NewPCG(1, 2)) feeds the Go program.
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

// prefix returns P with len+1 entries: P[0] = 0, P[i+1] = P[i] + a[i].
// So P[i] is the sum of the first i elements, and a[l..r] sums to P[r] - P[l].
fn prefix(a: &[i64]) -> Vec<i64> {
    std::iter::once(0)
        .chain(a.iter().scan(0, |s, &v| {
            *s += v;
            Some(*s)
        }))
        .collect()
}

// range_sum is the query: two reads. A reversed range (l > r) or one past the
// end (r > n) gives None instead of a panic or a wrong number.
fn range_sum(p: &[i64], l: usize, r: usize) -> Option<i64> {
    if l > r {
        return None;
    }
    Some(p.get(r)? - p.get(l)?)
}

// direct sums a[l..r] by looping: the oracle, and the cost the prefix array avoids.
fn direct(a: &[i64], l: usize, r: usize) -> i64 {
    a[l..r].iter().sum()
}

// rand_range returns a random half-open range 0 <= l <= r <= n.
fn rand_range(rng: &mut Pcg, n: usize) -> (usize, usize) {
    let l = rng.int_n(n as u64 + 1) as usize;
    let r = rng.int_n(n as u64 + 1) as usize;
    (l.min(r), l.max(r))
}

// Go prints a slice as [3 1 4]; Rust's {:?} prints [3, 1, 4]. Format by hand.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn main() {
    println!("== (a) the formula ==");
    let a = [3i64, 1, 4, 1, 5, 9, 2, 6];
    let p = prefix(&a);
    println!("  a = {}", go_slice(&a));
    println!("  P = {}", go_slice(&p));
    for (l, r) in [(0, 8), (2, 5), (3, 3), (7, 8)] {
        println!(
            "  a[{l}:{r}] = P[{r}] - P[{l}] = {} - {} = {}   (direct sum: {})",
            p[r],
            p[l],
            p[r] - p[l],
            direct(&a, l, r)
        );
    }
    println!(
        "  (extra, Rust only: range_sum(&p, 5, 2) = {:?}, range_sum(&p, 2, 9) = {:?}, range_sum(&p, 2, 5) = {:?})",
        range_sum(&p, 5, 2),
        range_sum(&p, 2, 9),
        range_sum(&p, 2, 5)
    );

    println!();
    println!("== (b) the off-by-one, on every range 0 <= l <= r <= 8 ==");
    // inc[i] = a[0]+...+a[i] has only n entries: no sentinel slot.
    let inc: Vec<i64> = a
        .iter()
        .scan(0, |s, &v| {
            *s += v;
            Some(*s)
        })
        .collect();
    let n = a.len();
    // None means "an index was out of range". The check comes BEFORE the read.
    // With usize there is no inc[-1]: `l - 1` is checked_sub, which says None.
    type Eval<'a> = &'a dyn Fn(usize, usize) -> Option<i64>;
    let no_sentinel = |l: usize, r: usize| Some(inc[r.checked_sub(1)?] - inc[l.checked_sub(1)?]);
    let same_array = |l: usize, r: usize| Some(inc.get(r)? - inc[l]);
    let sentinel = |l: usize, r: usize| Some(p[r] - p[l]);
    let variants: [(&str, Eval); 3] = [
        ("inc[r-1] - inc[l-1], no sentinel", &no_sentinel),
        ("inc[r] - inc[l], same array", &same_array),
        ("P[r] - P[l], sentinel P[0] = 0", &sentinel),
    ];
    println!(
        "  {:<34} {:>7} {:>6} {:>6} {:>13} {:>9}",
        "formula", "ranges", "right", "wrong", "out of range", "non-empty"
    );
    for (name, eval) in variants {
        let (mut total, mut right, mut wrong, mut out, mut right_non_empty) = (0, 0, 0, 0, 0);
        for l in 0..=n {
            for r in l..=n {
                total += 1;
                match eval(l, r) {
                    None => out += 1,
                    Some(sum) if sum == direct(&a, l, r) => {
                        right += 1;
                        if l < r {
                            right_non_empty += 1;
                        }
                    }
                    Some(_) => wrong += 1,
                }
            }
        }
        println!("  {name:<34} {total:>7} {right:>6} {wrong:>6} {out:>13} {right_non_empty:>9}");
    }
    println!(
        "  (last column: how many of the right answers were for non-empty ranges; there are 36 of those)"
    );

    println!();
    println!("== (c) element reads: re-summing vs a prefix array, n = 100000 ==");
    let mut rng = Pcg::new(1, 2);
    const N: usize = 100_000;
    let big: Vec<i64> = (0..N).map(|_| rng.int_n(1000) as i64 - 500).collect();
    let pb = prefix(&big);
    let ranges: Vec<(usize, usize)> = (0..N).map(|_| rand_range(&mut rng, N)).collect();
    // First 1000 queries for real: the answers agree, and the elements we
    // actually add are the sum of (r-l) that the table below counts arithmetically.
    let (mut agree, mut looped, mut formula) = (true, 0usize, 0usize);
    for &(l, r) in &ranges[..1000] {
        let mut s = 0;
        for &v in &big[l..r] {
            s += v;
            looped += 1;
        }
        formula += r - l;
        agree = agree && Some(s) == range_sum(&pb, l, r);
    }
    println!(
        "  first 1000 queries: answers agree {agree}, elements actually added {looped}, sum of lengths {formula}"
    );
    println!(
        "  {:>8} {:>16} {:>14}  cheaper",
        "q", "re-sum reads", "prefix reads"
    );
    let (mut sum, mut first_win, mut last_loss) = (0usize, 0, 0);
    for q in 1..=N {
        let (l, r) = ranges[q - 1];
        sum += r - l;
        let prefix_reads = N + 2 * q; // N to build, then two reads per query
        if prefix_reads < sum && first_win == 0 {
            first_win = q;
        }
        if sum <= prefix_reads {
            last_loss = q;
        }
        if [1, 2, 3, 5, 10, 100, 1000, 100000].contains(&q) {
            let who = if sum <= prefix_reads {
                "re-sum"
            } else {
                "prefix"
            };
            println!("  {q:>8} {sum:>16} {prefix_reads:>14}  {who}");
        }
    }
    println!(
        "  the prefix array first wins at q = {first_win}; re-summing was last ahead at q = {last_loss}"
    );

    println!();
    println!("== (d) 1000 interleaved update + query pairs at n = 100000 ==");
    // Each pair: a[i] += delta, then ask a range sum. Three ways to answer it.
    let mut work = big.clone(); // plain array, updated in place
    let mut p_full = prefix(&big); // rebuilt from scratch after every update
    let mut p_part = prefix(&big); // only P[i+1:] repaired after every update
    let (mut reads_full, mut reads_part, mut reads_sum, mut wrong) = (0usize, 0usize, 0usize, 0);
    for _ in 0..1000 {
        let i = rng.int_n(N as u64) as usize;
        let delta = rng.int_n(21) as i64 - 10;
        let (l, r) = rand_range(&mut rng, N);
        work[i] += delta;

        let mut acc = 0;
        for (j, &v) in work.iter().enumerate() {
            // rebuild everything: N reads
            acc += v;
            p_full[j + 1] = acc;
            reads_full += 1;
        }
        let mut acc = p_part[i];
        for (j, &v) in work.iter().enumerate().skip(i) {
            // P[..=i] is still valid: N-i reads
            acc += v;
            p_part[j + 1] = acc;
            reads_part += 1;
        }
        let mut s = 0;
        for &v in &work[l..r] {
            // no prefix array at all: r-l reads
            s += v;
            reads_sum += 1;
        }
        reads_full += 2;
        reads_part += 2;
        if p_full[r] - p_full[l] != s || p_part[r] - p_part[l] != s {
            wrong += 1;
        }
    }
    println!(
        "  {:<34} {:>14} {:>9}",
        "strategy", "element reads", "vs re-sum"
    );
    println!(
        "  {:<34} {reads_full:>14} {:>8.1}x",
        "rebuild all of P after each update",
        reads_full as f64 / reads_sum as f64
    );
    println!(
        "  {:<34} {reads_part:>14} {:>8.1}x",
        "repair only P[i+1:] after each one",
        reads_part as f64 / reads_sum as f64
    );
    println!(
        "  {:<34} {reads_sum:>14} {:>8.1}x",
        "no prefix array, re-sum the range", 1.0
    );
    println!("  wrong answers: {wrong}. A prefix array is for data that does not change;");
    println!("  for updates plus range sums the structure is a Fenwick tree (not covered here).");
}
```

**Rust output:**

```
== (a) the formula ==
  a = [3 1 4 1 5 9 2 6]
  P = [0 3 4 8 9 14 23 25 31]
  a[0:8] = P[8] - P[0] = 31 - 0 = 31   (direct sum: 31)
  a[2:5] = P[5] - P[2] = 14 - 4 = 10   (direct sum: 10)
  a[3:3] = P[3] - P[3] = 8 - 8 = 0   (direct sum: 0)
  a[7:8] = P[8] - P[7] = 31 - 25 = 6   (direct sum: 6)
  (extra, Rust only: range_sum(&p, 5, 2) = None, range_sum(&p, 2, 9) = None, range_sum(&p, 2, 5) = Some(10))

== (b) the off-by-one, on every range 0 <= l <= r <= 8 ==
  formula                             ranges  right  wrong  out of range non-empty
  inc[r-1] - inc[l-1], no sentinel        45     36      0             9        28
  inc[r] - inc[l], same array             45      9     27             9         1
  P[r] - P[l], sentinel P[0] = 0          45     45      0             0        36
  (last column: how many of the right answers were for non-empty ranges; there are 36 of those)

== (c) element reads: re-summing vs a prefix array, n = 100000 ==
  first 1000 queries: answers agree true, elements actually added 33220513, sum of lengths 33220513
         q     re-sum reads   prefix reads  cheaper
         1            45695         100002  re-sum
         2            61590         100004  re-sum
         3           148620         100006  prefix
         5           241650         100010  prefix
        10           464009         100020  prefix
       100          3367651         100200  prefix
      1000         33220513         102000  prefix
    100000       3332758140         300000  prefix
  the prefix array first wins at q = 3; re-summing was last ahead at q = 2

== (d) 1000 interleaved update + query pairs at n = 100000 ==
  strategy                            element reads vs re-sum
  rebuild all of P after each update      100002000      2.9x
  repair only P[i+1:] after each one       50732343      1.5x
  no prefix array, re-sum the range        34099009      1.0x
  wrong answers: 0. A prefix array is for data that does not change;
  for updates plus range sums the structure is a Fenwick tree (not covered here).
```

> **Go vs Rust:** Everything printed is identical to Go (same PCG stream; the extra `range_sum` line is Rust only). `P` is built with `iter::once(0).chain(scan(..))`, and the query returns `Option<i64>`: `None` for `l > r` or `r > n` via `get`. In the sentinel-free variant, "index −1" cannot even be written with a `usize`: `l.checked_sub(1)?` turns it into `None`, which is the 9 "out of range" ranges, where Go needs an explicit `l-1 < 0` test before reading (or it panics).

**Complexity:** build Θ(n) time and space, query Θ(1), and a point update costs Θ(n) to repair. Measured at n = 100,000: re-summing was cheaper at q = 1 and 2, and the prefix array first won at q = 3 (n + 2q reads against the sum of the range lengths, about n/3 per random range); in the update workload, repairing `P[i+1:]` after each of 1,000 updates cost 1.5x the reads of simply re-summing, and rebuilding all of P cost 2.9x. The leading zero and the half-open range remove the off-by-one: without them 9 of the 45 ranges could not even be evaluated.

---

## 5. Prefix sums that overflow, and subtractions that underflow

`🟢 easy` · *Integer overflow, usize underflow*

Two separate hazards live in the same subtraction `P[r] − P[l]`. **Overflow:** `P[n]` can pass the integer type even when every element and every range answer fits; wrapping is arithmetic modulo 2⁶⁴, so the subtraction is *still exact whenever the true sum fits* — the overflow cancels. **Underflow:** with unsigned prefixes, a reversed range (`l > r`) wraps to a huge number. Prefix sums stay Θ(n) to build and Θ(1) to query; what changes is what the answer *means*, and the program checks that on every range of a 20-element array.

**Steps:**

1. Sum 20 values near 2⁵⁹ into signed prefixes, print them, and find the first index where P goes negative.
2. Compare every range `P[r] − P[l]` with an exact big-integer sum, grouped by range length, and test "exact if and only if the true sum fits".
3. Switch to unsigned prefixes for the reversed range, and write the checked add that Go does not give you.

```go
package main

import (
	"fmt"
	"math"
	"math/big"
	"math/rand/v2"
)

// checkedAdd reports whether a+b fits in an int. Go has no checked add in the
// standard library, and ordinary + never tells you: signed overflow wraps
// (the spec: it "does not cause a run-time panic"), in every build.
func checkedAdd(a, b int) (int, bool) {
	if (b > 0 && a > math.MaxInt-b) || (b < 0 && a < math.MinInt-b) {
		return 0, false
	}
	return a + b, true
}

func main() {
	rng := rand.New(rand.NewPCG(1, 2))
	const base = int(1) << 59
	a := make([]int, 20)
	for i := range a {
		a[i] = base + rng.IntN(1<<20)
	}
	p := make([]int, len(a)+1) // P[0] = 0, P[i+1] = P[i] + a[i], wrapping silently
	for i, v := range a {
		p[i+1] = p[i] + v
	}

	fmt.Println("== (a) 20 values 2^59 + IntN(1<<20): the prefix sums, as signed ints ==")
	firstNeg := -1
	for i, v := range p {
		if v < 0 && firstNeg < 0 {
			firstNeg = i
		}
		fmt.Printf("  P[%2d] = %20d\n", i, v)
	}
	fmt.Printf("  first negative: P[%d]; expected 2^63 / 2^59 = %d\n", firstNeg, 1<<(63-59))

	fmt.Println()
	fmt.Println("== (b) every range l < r against an exact big.Int sum ==")
	two64 := new(big.Int).Lsh(big.NewInt(1), 64)
	type group struct{ ranges, wrapped, wrong, offBy2_64 int }
	var short, long group // lengths <= 15, lengths >= 16
	iff := true
	for l := 0; l < len(p); l++ {
		for r := l + 1; r < len(p); r++ {
			exact := new(big.Int)
			for _, v := range a[l:r] {
				exact.Add(exact, big.NewInt(int64(v)))
			}
			got := p[r] - p[l] // wraps if it must
			diff := new(big.Int).Sub(exact, big.NewInt(int64(got)))
			g := &short
			if r-l >= 16 {
				g = &long
			}
			g.ranges++
			if p[r] < 0 { // P[r] itself wrapped (every prefix from P[firstNeg] on did)
				g.wrapped++
			}
			if diff.Sign() != 0 {
				g.wrong++
				if diff.Cmp(two64) == 0 {
					g.offBy2_64++
				}
			}
			iff = iff && (diff.Sign() == 0) == exact.IsInt64()
		}
	}
	fmt.Printf("  %-12s %7s %12s %9s %20s\n", "range length", "ranges", "P[r] wrapped", "mismatch", "exact - got == 2^64")
	fmt.Printf("  %-12s %7d %12d %9d %20d\n", "1 to 15", short.ranges, short.wrapped, short.wrong, short.offBy2_64)
	fmt.Printf("  %-12s %7d %12d %9d %20d\n", "16 to 20", long.ranges, long.wrapped, long.wrong, long.offBy2_64)
	fmt.Printf("  answer exact if and only if the true sum fits in an int64: %v (all %d ranges)\n",
		iff, short.ranges+long.ranges)

	fmt.Println()
	fmt.Println("== (c) unsigned prefixes: the reversed range ==")
	small := []uint64{3, 1, 4, 1, 5, 9, 2, 6}
	pu := make([]uint64, len(small)+1)
	for i, v := range small {
		pu[i+1] = pu[i] + v
	}
	l, r := 5, 2 // reversed: l > r
	wrapped := pu[r] - pu[l]
	fmt.Printf("  P[%d] - P[%d] = %d - %d = %d\n", r, l, pu[r], pu[l], wrapped)
	fmt.Printf("  the same bits read as int64: %d (wrapping is modulo 2^64, so it is reversible)\n", int64(wrapped))
	fmt.Printf("  the symmetric answer, written by hand: %d\n", pu[l]-pu[r])
	n := uint(len(small[:3]))
	fmt.Printf("  and uint(len(s)) - 7 for a 3-element slice: %d\n", n-7)

	fmt.Println()
	fmt.Println("== (d) the same sums with a checked add ==")
	fmt.Println("  Go has no debug/release difference in arithmetic: + wraps in every build.")
	sum, firstFail := 0, -1
	for i, v := range a {
		if _, ok := checkedAdd(sum, v); !ok && firstFail < 0 {
			firstFail = i + 1 // the prefix P[i+1] is the one that overflowed
		}
		sum += v
	}
	fmt.Printf("  plain + over the 20 values: %d (equals P[20]: %v)\n", sum, sum == p[20])
	fmt.Printf("  checkedAdd (hand-written): first failure while computing P[%d]\n", firstFail)
}
```

**Output:**

```
== (a) 20 values 2^59 + IntN(1<<20): the prefix sums, as signed ints ==
  P[ 0] =                    0
  P[ 1] =   576460752304403728
  P[ 2] =  1152921504608739708
  P[ 3] =  1729382256913179140
  P[ 4] =  2305843009217307550
  P[ 5] =  2882303761521351918
  P[ 6] =  3458764513825322922
  P[ 7] =  4035225266129586723
  P[ 8] =  4611686018433808817
  P[ 9] =  5188146770737276746
  P[10] =  5764607523040738298
  P[11] =  6341068275344386281
  P[12] =  6917529027647992115
  P[13] =  7493989779951699182
  P[14] =  8070450532256048011
  P[15] =  8646911284559738089
  P[16] = -9223372036845729180
  P[17] = -8646911284541639861
  P[18] = -8070450532237565843
  P[19] = -7493989779933829722
  P[20] = -6917529027630114540
  first negative: P[16]; expected 2^63 / 2^59 = 16

== (b) every range l < r against an exact big.Int sum ==
  range length  ranges P[r] wrapped  mismatch  exact - got == 2^64
  1 to 15          195           75         0                    0
  16 to 20          15           15        15                   15
  answer exact if and only if the true sum fits in an int64: true (all 210 ranges)

== (c) unsigned prefixes: the reversed range ==
  P[2] - P[5] = 4 - 14 = 18446744073709551606
  the same bits read as int64: -10 (wrapping is modulo 2^64, so it is reversible)
  the symmetric answer, written by hand: 10
  and uint(len(s)) - 7 for a 3-element slice: 18446744073709551612

== (d) the same sums with a checked add ==
  Go has no debug/release difference in arithmetic: + wraps in every build.
  plain + over the 20 values: -6917529027630114540 (equals P[20]: true)
  checkedAdd (hand-written): first failure while computing P[16]
```

**Rust:**

```rust
// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) feeds this program the same numbers that Go's
// rand.New(rand.NewPCG(1, 2)) feeds the Go program.
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

fn main() {
    let mut rng = Pcg::new(1, 2);
    const BASE: i64 = 1 << 59;
    let a: Vec<i64> = (0..20).map(|_| BASE + rng.int_n(1 << 20) as i64).collect();
    // P[0] = 0, P[i+1] = P[i] + a[i]. The wrapping is written out: plain `+`
    // would wrap under -O (what Go does everywhere) but panic in a debug build.
    let mut p = vec![0i64; a.len() + 1];
    for (i, &v) in a.iter().enumerate() {
        p[i + 1] = p[i].wrapping_add(v);
    }

    println!("== (a) 20 values 2^59 + IntN(1<<20): the prefix sums, as signed ints ==");
    let mut first_neg = None;
    for (i, &v) in p.iter().enumerate() {
        if v < 0 && first_neg.is_none() {
            first_neg = Some(i);
        }
        println!("  P[{i:>2}] = {v:>20}");
    }
    println!(
        "  first negative: P[{}]; expected 2^63 / 2^59 = {}",
        first_neg.map_or("none".to_string(), |i| i.to_string()),
        1u32 << (63 - 59)
    );

    println!();
    println!("== (b) every range l < r against an exact i128 sum ==");
    const TWO64: i128 = 1 << 64;
    #[derive(Default)]
    struct Group {
        ranges: usize,
        wrapped: usize,
        wrong: usize,
        off_by_2_64: usize,
    }
    let (mut short, mut long) = (Group::default(), Group::default()); // lengths <= 15, >= 16
    let mut iff = true;
    for l in 0..p.len() {
        for r in l + 1..p.len() {
            let exact: i128 = a[l..r].iter().map(|&v| i128::from(v)).sum();
            let got = p[r].wrapping_sub(p[l]); // wraps if it must
            let diff = exact - i128::from(got);
            let g = if r - l >= 16 { &mut long } else { &mut short };
            g.ranges += 1;
            if p[r] < 0 {
                // P[r] itself wrapped (every prefix from P[first_neg] on did)
                g.wrapped += 1;
            }
            if diff != 0 {
                g.wrong += 1;
                if diff == TWO64 {
                    g.off_by_2_64 += 1;
                }
            }
            iff = iff && (diff == 0) == i64::try_from(exact).is_ok();
        }
    }
    println!(
        "  {:<12} {:>7} {:>12} {:>9} {:>20}",
        "range length", "ranges", "P[r] wrapped", "mismatch", "exact - got == 2^64"
    );
    for (name, g) in [("1 to 15", &short), ("16 to 20", &long)] {
        println!(
            "  {name:<12} {:>7} {:>12} {:>9} {:>20}",
            g.ranges, g.wrapped, g.wrong, g.off_by_2_64
        );
    }
    println!(
        "  answer exact if and only if the true sum fits in an int64: {iff} (all {} ranges)",
        short.ranges + long.ranges
    );

    println!();
    println!("== (c) unsigned prefixes: the reversed range ==");
    let small = [3u64, 1, 4, 1, 5, 9, 2, 6];
    let mut pu = vec![0u64; small.len() + 1];
    for (i, &v) in small.iter().enumerate() {
        pu[i + 1] = pu[i] + v;
    }
    let (l, r) = (5, 2); // reversed: l > r
    println!("  P[{r}] - P[{l}] = {} - {}:", pu[r], pu[l]);
    println!("    checked_sub  -> {:?}", pu[r].checked_sub(pu[l]));
    let wrapped = pu[r].wrapping_sub(pu[l]);
    println!("    wrapping_sub -> {wrapped}  (the number Go prints)");
    println!(
        "    the same bits read as i64: {} (wrapping is modulo 2^64, so it is reversible)",
        wrapped as i64
    );
    println!(
        "    abs_diff     -> {}  (the symmetric answer)",
        pu[r].abs_diff(pu[l])
    );
    let n = small[..3].len();
    println!(
        "  and len - 7 for a 3-element slice: checked_sub -> {:?}, wrapping_sub -> {}",
        n.checked_sub(7),
        n.wrapping_sub(7)
    );

    println!();
    println!("== (d) the same sums with a checked add ==");
    println!(
        "  built with debug assertions (overflow checks on): {}",
        cfg!(debug_assertions)
    );
    let mut sum = 0i64;
    let mut first_fail = None;
    for (i, &v) in a.iter().enumerate() {
        if sum.checked_add(v).is_none() && first_fail.is_none() {
            first_fail = Some(i + 1); // the prefix P[i+1] is the one that overflowed
        }
        sum = sum.wrapping_add(v);
    }
    println!(
        "  wrapping_add over the 20 values: {sum} (equals P[20]: {})",
        sum == p[20]
    );
    println!(
        "  checked_add: first None while computing P[{}]",
        first_fail.map_or("none".to_string(), |i| i.to_string())
    );
    if std::env::args().any(|arg| arg == "--panic") {
        // Iterator::sum is compiled with the caller's overflow checks:
        // -O wraps and prints a number, a build without -O panics here.
        let total: i64 = a.iter().sum();
        println!("  Iterator::sum over the 20 values: {total}");
    } else {
        println!(
            "  (Iterator::sum is not run by default: pass --panic to apply it to these 20 values)"
        );
    }
}
```

**Run it (Rust):**

```bash
rustc -O --edition 2024 main.rs && ./main            # add --panic to run Iterator::sum on the same 20 values
```

**Rust output:**

```
== (a) 20 values 2^59 + IntN(1<<20): the prefix sums, as signed ints ==
  P[ 0] =                    0
  P[ 1] =   576460752304403728
  P[ 2] =  1152921504608739708
  P[ 3] =  1729382256913179140
  P[ 4] =  2305843009217307550
  P[ 5] =  2882303761521351918
  P[ 6] =  3458764513825322922
  P[ 7] =  4035225266129586723
  P[ 8] =  4611686018433808817
  P[ 9] =  5188146770737276746
  P[10] =  5764607523040738298
  P[11] =  6341068275344386281
  P[12] =  6917529027647992115
  P[13] =  7493989779951699182
  P[14] =  8070450532256048011
  P[15] =  8646911284559738089
  P[16] = -9223372036845729180
  P[17] = -8646911284541639861
  P[18] = -8070450532237565843
  P[19] = -7493989779933829722
  P[20] = -6917529027630114540
  first negative: P[16]; expected 2^63 / 2^59 = 16

== (b) every range l < r against an exact i128 sum ==
  range length  ranges P[r] wrapped  mismatch  exact - got == 2^64
  1 to 15          195           75         0                    0
  16 to 20          15           15        15                   15
  answer exact if and only if the true sum fits in an int64: true (all 210 ranges)

== (c) unsigned prefixes: the reversed range ==
  P[2] - P[5] = 4 - 14:
    checked_sub  -> None
    wrapping_sub -> 18446744073709551606  (the number Go prints)
    the same bits read as i64: -10 (wrapping is modulo 2^64, so it is reversible)
    abs_diff     -> 10  (the symmetric answer)
  and len - 7 for a 3-element slice: checked_sub -> None, wrapping_sub -> 18446744073709551612

== (d) the same sums with a checked add ==
  built with debug assertions (overflow checks on): false
  wrapping_add over the 20 values: -6917529027630114540 (equals P[20]: true)
  checked_add: first None while computing P[16]
  (Iterator::sum is not run by default: pass --panic to apply it to these 20 values)
```

> **Go vs Rust:** Sections (a) and (b) print the same numbers as Go (`i128` plays the part of `math/big`); (c) and (d) differ in form because Rust offers the policy as methods: `checked_sub` gives `None` where Go has no way to ask, `wrapping_sub` prints the number Go prints, `abs_diff` the symmetric answer. The whole Rust point is that plain `+` wraps under `-O` but panics in a debug build, so every wrap here is written `wrapping_add` and the default run cannot panic in either build. Run separately and not shipped: `Iterator::sum` over these 20 values printed -6917529027630114540 with `-O` and, built without `-O`, `./main --panic` panicked with "attempt to add with overflow" inside `core/src/iter/traits/accum.rs` (that fold carries `#[rustc_inherit_overflow_checks]`, read in the source). Go has no such split: signed overflow wraps in every build.

**Complexity:** unchanged: Θ(n) build, Θ(1) query. Correctness is a property of the arithmetic: of the 210 ranges, the 195 of length ≤ 15 were exact (75 of them computed from a wrapped `P[r]`), and the 15 of length ≥ 16 each came out exactly 2⁶⁴ below the true sum — exact if and only if the true sum fits in an `int64`. A reversed range on unsigned prefixes is a bug, not a result: it printed 18446744073709551606 where the symmetric answer is 10.

---

## 6. Difference arrays: range add in Θ(1)

`🟢 easy` · *Difference arrays*

The inverse of prefix sums. "Add `v` to `a[l:r]`" becomes two writes, `D[l] += v` and `D[r] -= v`, on an array D of length n + 1 (so that `r == n` has a slot to write to); one running sum over D then rebuilds the array. After m updates and one read the cost is Θ(n + m) instead of Θ(Σ lengths) — but the rebuild is a fixed Θ(n), so with very few updates the naive loop is cheaper, and the program finds the crossover by counting writes.

**Steps:**

1. Apply three range adds to an 8-element array, printing D after each one, then rebuild with a running sum and compare with the naive array.
2. Count writes for m = 1 to 10,000 random range adds at n = 100,000, and find the m where the difference array starts to win.
3. Check that `diff` and `prefix` undo each other on 5,000 random arrays, then use a difference array to find the deepest overlap of 1,000 intervals.

```go
package main

import (
	"fmt"
	"math/rand/v2"
	"slices"
)

// rangeAdd records "add v to a[l:r]" in the difference array d, in TWO writes.
// d has n+1 slots so that r == n (a range running to the end) has somewhere
// to write. A range outside 0 <= l <= r <= n is rejected, not panicked on.
func rangeAdd(d []int, l, r, v int) bool {
	if l < 0 || l > r || r >= len(d) {
		return false
	}
	d[l] += v
	d[r] -= v
	return true
}

// materialise rebuilds the array from d: a running sum, n steps. The extra
// slot d[n] is never read.
func materialise(d []int) []int {
	a := make([]int, len(d)-1)
	s := 0
	for i := range a {
		s += d[i]
		a[i] = s
	}
	return a
}

// diff and runningSum are inverse bijections on length-n arrays:
// diff(a)[i] = a[i] - a[i-1] (with a[-1] = 0), runningSum(a)[i] = a[0]+...+a[i]
// (that is P[1:] of example 4).
func diff(a []int) []int {
	d := make([]int, len(a))
	for i, v := range a {
		d[i] = v
		if i > 0 {
			d[i] -= a[i-1]
		}
	}
	return d
}

func runningSum(a []int) []int {
	out := make([]int, len(a))
	s := 0
	for i, v := range a {
		s += v
		out[i] = s
	}
	return out
}

func randRange(rng *rand.Rand, n int) (l, r int) {
	l, r = rng.IntN(n+1), rng.IntN(n+1)
	if l > r {
		l, r = r, l
	}
	return l, r
}

func main() {
	fmt.Println("== (a) n = 8, three range adds ==")
	const n = 8
	d := make([]int, n+1)
	naive := make([]int, n)
	fmt.Printf("  %-22s %v\n", "D at the start:", d)
	for _, u := range [][3]int{{1, 4, 2}, {3, 7, 3}, {0, 2, -1}, {5, 9, 1}} {
		l, r, v := u[0], u[1], u[2]
		if !rangeAdd(d, l, r, v) {
			fmt.Printf("  add(%d,%d,%+d): rejected, r = %d > n = %d\n", l, r, v, r, n)
			continue
		}
		for i := l; i < r; i++ {
			naive[i] += v
		}
		fmt.Printf("  %-22s %v\n", fmt.Sprintf("D after add(%d,%d,%+d):", l, r, v), d)
	}
	got := materialise(d)
	fmt.Printf("  %-22s %v\n", "materialise(D):", got)
	fmt.Printf("  %-22s %v  equal: %v\n", "naive array:", naive, slices.Equal(got, naive))

	fmt.Println()
	fmt.Println("== (b) writes for m random range adds, n = 100000 ==")
	rng := rand.New(rand.NewPCG(1, 2))
	const N, M = 100_000, 10_000
	type update struct{ l, r, v int }
	updates := make([]update, M)
	for i := range updates {
		l, r := randRange(rng, N)
		updates[i] = update{l, r, rng.IntN(11) - 5}
	}
	big, bigD := make([]int, N), make([]int, N+1)
	checkpoints := map[int]bool{1: true, 2: true, 3: true, 10: true, 100: true, 1000: true, 10000: true}
	naiveWrites, diffWrites, firstWin, lastLoss := 0, 0, 0, 0
	fmt.Printf("  %6s %14s %14s  %-8s %s\n", "m", "naive writes", "diff writes", "cheaper", "arrays equal")
	for m := 1; m <= M; m++ {
		u := updates[m-1]
		for i := u.l; i < u.r; i++ { // the naive way: one write per element
			big[i] += u.v
			naiveWrites++
		}
		rangeAdd(bigD, u.l, u.r, u.v) // the difference way: two writes
		diffWrites += 2
		total := diffWrites + N // plus N writes to rebuild once
		if total < naiveWrites && firstWin == 0 {
			firstWin = m
		}
		if naiveWrites <= total {
			lastLoss = m
		}
		if checkpoints[m] {
			who := "diff"
			if naiveWrites <= total {
				who = "naive"
			}
			fmt.Printf("  %6d %14d %14d  %-8s %v\n", m, naiveWrites, total, who, slices.Equal(big, materialise(bigD)))
		}
	}
	fmt.Printf("  the difference array first wins at m = %d; naive was last ahead at m = %d\n", firstWin, lastLoss)

	fmt.Println()
	fmt.Println("== (c) diff and prefix undo each other: 5000 random arrays ==")
	bad1, bad2 := 0, 0
	for range 5000 {
		a := make([]int, rng.IntN(21))
		for i := range a {
			a[i] = rng.IntN(11) - 5
		}
		if !slices.Equal(runningSum(diff(a)), a) {
			bad1++
		}
		if !slices.Equal(diff(runningSum(a)), a) {
			bad2++
		}
	}
	fmt.Printf("  runningSum(diff(a)) != a: %d cases; diff(runningSum(a)) != a: %d cases\n", bad1, bad2)

	fmt.Println()
	fmt.Println("== (d) deepest overlap of 1000 intervals on [0, 5000) ==")
	const W, K = 5000, 1000
	type interval struct{ l, r int }
	ivs := make([]interval, K)
	cov := make([]int, W+1)
	for i := range ivs {
		l := rng.IntN(W)
		r := min(l+1+rng.IntN(500), W)
		ivs[i] = interval{l, r}
		rangeAdd(cov, l, r, 1) // two writes each
	}
	maxD, atD, run := 0, 0, 0
	for x := 0; x < W; x++ { // one scan: the running sum is the depth at x
		run += cov[x]
		if run > maxD {
			maxD, atD = run, x
		}
	}
	maxB, atB, tests := 0, 0, 0
	for x := 0; x < W; x++ { // brute force: ask every interval about every point
		depth := 0
		for _, iv := range ivs {
			tests++
			if iv.l <= x && x < iv.r {
				depth++
			}
		}
		if depth > maxB {
			maxB, atB = depth, x
		}
	}
	fmt.Printf("  difference array: max %d first at x = %d; brute force: max %d first at x = %d; equal: %v\n",
		maxD, atD, maxB, atB, maxD == maxB && atD == atB)
	fmt.Printf("  work: difference array %d writes + %d scan steps; brute force %d interval tests\n", 2*K, W, tests)
}
```

**Output:**

```
== (a) n = 8, three range adds ==
  D at the start:        [0 0 0 0 0 0 0 0 0]
  D after add(1,4,+2):   [0 2 0 0 -2 0 0 0 0]
  D after add(3,7,+3):   [0 2 0 3 -2 0 0 -3 0]
  D after add(0,2,-1):   [-1 2 1 3 -2 0 0 -3 0]
  add(5,9,+1): rejected, r = 9 > n = 8
  materialise(D):        [-1 1 2 5 3 3 3 0]
  naive array:           [-1 1 2 5 3 3 3 0]  equal: true

== (b) writes for m random range adds, n = 100000 ==
       m   naive writes    diff writes  cheaper  arrays equal
       1          15294         100002  naive    true
       2          71527         100004  naive    true
       3          76580         100006  naive    true
      10         331502         100020  diff     true
     100        3124752         100200  diff     true
    1000       32252877         102000  diff     true
   10000      332141429         120000  diff     true
  the difference array first wins at m = 4; naive was last ahead at m = 3

== (c) diff and prefix undo each other: 5000 random arrays ==
  runningSum(diff(a)) != a: 0 cases; diff(runningSum(a)) != a: 0 cases

== (d) deepest overlap of 1000 intervals on [0, 5000) ==
  difference array: max 64 first at x = 1878; brute force: max 64 first at x = 1878; equal: true
  work: difference array 2000 writes + 5000 scan steps; brute force 5000000 interval tests
```

**Rust:**

```rust
// Pcg: a port of Go's math/rand/v2 PCG (PCG-DXSM, 128-bit state) and of its
// IntN, so the same seed (1, 2) feeds this program the same numbers that Go's
// rand.New(rand.NewPCG(1, 2)) feeds the Go program.
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

// range_add records "add v to a[l..r]" in the difference array d, in TWO
// writes. d has n+1 slots so that r == n (a range running to the end) has
// somewhere to write. A range outside 0 <= l <= r <= n is rejected (false):
// d[r] with r > n would be a bounds-check panic in Rust.
fn range_add(d: &mut [i64], l: usize, r: usize, v: i64) -> bool {
    if l > r || r >= d.len() {
        return false;
    }
    d[l] += v;
    d[r] -= v;
    true
}

// materialise rebuilds the array from d: a running sum, n steps. The extra
// slot d[n] is never read.
fn materialise(d: &[i64]) -> Vec<i64> {
    running_sum(&d[..d.len() - 1])
}

// diff and running_sum are inverse bijections on length-n arrays:
// diff(a)[i] = a[i] - a[i-1] (with a[-1] = 0), running_sum(a)[i] = a[0]+...+a[i]
// (that is P[1:] of example 4).
fn diff(a: &[i64]) -> Vec<i64> {
    // zip pairs each element with its predecessor; the first has none, so 0.
    a.iter()
        .zip(std::iter::once(&0).chain(a.iter()))
        .map(|(cur, prev)| cur - prev)
        .collect()
}

fn running_sum(a: &[i64]) -> Vec<i64> {
    a.iter()
        .scan(0, |s, &v| {
            *s += v;
            Some(*s)
        })
        .collect()
}

fn rand_range(rng: &mut Pcg, n: usize) -> (usize, usize) {
    let l = rng.int_n(n as u64 + 1) as usize;
    let r = rng.int_n(n as u64 + 1) as usize;
    (l.min(r), l.max(r))
}

// Go prints a slice as [1 3 4]; Rust's {:?} prints [1, 3, 4]. Format by hand.
fn go_slice(xs: &[i64]) -> String {
    let items: Vec<String> = xs.iter().map(i64::to_string).collect();
    format!("[{}]", items.join(" "))
}

fn main() {
    println!("== (a) n = 8, three range adds ==");
    const N8: usize = 8;
    let mut d = vec![0i64; N8 + 1];
    let mut naive = vec![0i64; N8];
    println!("  {:<22} {}", "D at the start:", go_slice(&d));
    for (l, r, v) in [(1, 4, 2), (3, 7, 3), (0, 2, -1), (5, 9, 1)] {
        if !range_add(&mut d, l, r, v) {
            println!("  add({l},{r},{v:+}): rejected, r = {r} > n = {N8}");
            continue;
        }
        for x in &mut naive[l..r] {
            *x += v;
        }
        println!(
            "  {:<22} {}",
            format!("D after add({l},{r},{v:+}):"),
            go_slice(&d)
        );
    }
    let got = materialise(&d);
    println!("  {:<22} {}", "materialise(D):", go_slice(&got));
    println!(
        "  {:<22} {}  equal: {}",
        "naive array:",
        go_slice(&naive),
        got == naive
    );

    println!();
    println!("== (b) writes for m random range adds, n = 100000 ==");
    let mut rng = Pcg::new(1, 2);
    const N: usize = 100_000;
    const M: usize = 10_000;
    let updates: Vec<(usize, usize, i64)> = (0..M)
        .map(|_| {
            let (l, r) = rand_range(&mut rng, N);
            (l, r, rng.int_n(11) as i64 - 5)
        })
        .collect();
    let (mut big, mut big_d) = (vec![0i64; N], vec![0i64; N + 1]);
    let (mut naive_writes, mut diff_writes, mut first_win, mut last_loss) = (0usize, 0usize, 0, 0);
    println!(
        "  {:>6} {:>14} {:>14}  {:<8} arrays equal",
        "m", "naive writes", "diff writes", "cheaper"
    );
    for m in 1..=M {
        let (l, r, v) = updates[m - 1];
        for x in &mut big[l..r] {
            // the naive way: one write per element
            *x += v;
            naive_writes += 1;
        }
        range_add(&mut big_d, l, r, v); // the difference way: two writes
        diff_writes += 2;
        let total = diff_writes + N; // plus N writes to rebuild once
        if total < naive_writes && first_win == 0 {
            first_win = m;
        }
        if naive_writes <= total {
            last_loss = m;
        }
        if [1, 2, 3, 10, 100, 1000, 10000].contains(&m) {
            let who = if naive_writes <= total {
                "naive"
            } else {
                "diff"
            };
            println!(
                "  {m:>6} {naive_writes:>14} {total:>14}  {who:<8} {}",
                big == materialise(&big_d)
            );
        }
    }
    println!(
        "  the difference array first wins at m = {first_win}; naive was last ahead at m = {last_loss}"
    );

    println!();
    println!("== (c) diff and prefix undo each other: 5000 random arrays ==");
    let (mut bad1, mut bad2) = (0, 0);
    for _ in 0..5000 {
        let len = rng.int_n(21) as usize;
        let a: Vec<i64> = (0..len).map(|_| rng.int_n(11) as i64 - 5).collect();
        if running_sum(&diff(&a)) != a {
            bad1 += 1;
        }
        if diff(&running_sum(&a)) != a {
            bad2 += 1;
        }
    }
    println!("  running_sum(diff(a)) != a: {bad1} cases; diff(running_sum(a)) != a: {bad2} cases");

    println!();
    println!("== (d) deepest overlap of 1000 intervals on [0, 5000) ==");
    const W: usize = 5000;
    const K: usize = 1000;
    let mut ivs = Vec::with_capacity(K);
    let mut cov = vec![0i64; W + 1];
    for _ in 0..K {
        let l = rng.int_n(W as u64) as usize;
        let r = (l + 1 + rng.int_n(500) as usize).min(W);
        ivs.push((l, r));
        range_add(&mut cov, l, r, 1); // two writes each
    }
    let (mut max_d, mut at_d, mut run) = (0, 0, 0);
    for (x, &c) in cov[..W].iter().enumerate() {
        // one scan: the running sum is the depth at x
        run += c;
        if run > max_d {
            (max_d, at_d) = (run, x);
        }
    }
    let (mut max_b, mut at_b, mut tests) = (0, 0, 0usize);
    for x in 0..W {
        // brute force: ask every interval about every point
        let mut depth = 0;
        for &(l, r) in &ivs {
            tests += 1;
            if l <= x && x < r {
                depth += 1;
            }
        }
        if depth > max_b {
            (max_b, at_b) = (depth, x);
        }
    }
    println!(
        "  difference array: max {max_d} first at x = {at_d}; brute force: max {max_b} first at x = {at_b}; equal: {}",
        max_d == max_b && at_d == at_b
    );
    println!(
        "  work: difference array {} writes + {W} scan steps; brute force {tests} interval tests",
        2 * K
    );
}
```

**Rust output:**

```
== (a) n = 8, three range adds ==
  D at the start:        [0 0 0 0 0 0 0 0 0]
  D after add(1,4,+2):   [0 2 0 0 -2 0 0 0 0]
  D after add(3,7,+3):   [0 2 0 3 -2 0 0 -3 0]
  D after add(0,2,-1):   [-1 2 1 3 -2 0 0 -3 0]
  add(5,9,+1): rejected, r = 9 > n = 8
  materialise(D):        [-1 1 2 5 3 3 3 0]
  naive array:           [-1 1 2 5 3 3 3 0]  equal: true

== (b) writes for m random range adds, n = 100000 ==
       m   naive writes    diff writes  cheaper  arrays equal
       1          15294         100002  naive    true
       2          71527         100004  naive    true
       3          76580         100006  naive    true
      10         331502         100020  diff     true
     100        3124752         100200  diff     true
    1000       32252877         102000  diff     true
   10000      332141429         120000  diff     true
  the difference array first wins at m = 4; naive was last ahead at m = 3

== (c) diff and prefix undo each other: 5000 random arrays ==
  running_sum(diff(a)) != a: 0 cases; diff(running_sum(a)) != a: 0 cases

== (d) deepest overlap of 1000 intervals on [0, 5000) ==
  difference array: max 64 first at x = 1878; brute force: max 64 first at x = 1878; equal: true
  work: difference array 2000 writes + 5000 scan steps; brute force 5000000 interval tests
```

> **Go vs Rust:** Identical to Go except that the function names are snake_case (the 5,000-array line says `running_sum`): same crossover (naive ahead through m = 3, difference array first wins at m = 4), same maximum overlap, same counts. Both languages would panic on `d[9]` for the rejected `add(5,9,+1)`, so both validate `r >= d.len()` first. The running sums are `scan`s, and `diff` pairs each element with its predecessor by zipping against `once(&0).chain(..)`, which is how Rust writes the "a[-1] = 0" that Go handles with an `if i > 0`.

**Complexity:** range add Θ(1) (two writes), materialise Θ(n), so m updates and one read cost Θ(n + m) against Θ(Σ lengths). Measured at n = 100,000: the naive loop was ahead through m = 3 (76,580 writes against 100,006) and the difference array first won at m = 4, because the rebuild is a fixed n; at m = 10,000 it was 120,000 writes against 332,141,429. `diff` and a running sum undid each other in all 5,000 random arrays, and the same machinery found a deepest overlap of 64 among 1,000 intervals with 7,000 operations instead of 5,000,000 tests.

---

> Next tier: [🟡 medium](2-medium.md) — the container with most water, the Dutch flag proved
> exhaustively, the two shapes of variable window, and 2-D prefix sums.

---
*Global progress: [../../PROGRESS.md](../../PROGRESS.md).*
